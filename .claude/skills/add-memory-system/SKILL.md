---
name: add-memory-system
description: Add a persistent memory system to NanoClaw. Agents can store facts, search past conversations, and recall information across sessions. Main group controls memory for all groups.
---

# Add Memory System

This skill adds a persistent memory system that lets agents remember facts, preferences, and conversation history across sessions. Memory is stored as markdown files, indexed into SQLite FTS, and searchable via MCP tools.

## Architecture Overview

```
groups/{folder}/memory/       ← Agent writes facts here directly
groups/{folder}/conversations/ ← Auto-archived on compact (pre-compact hook)
         ↓
   memory-indexer.ts          ← Host-side: scans .md files, chunks text, upserts to DB
         ↓
   store/messages.db          ← memory_chunks table + LIKE-based search (中文 friendly)
         ↓
   ipc-mcp-stdio.ts           ← Container-side: memory_search / memory_enable / memory_cleanup tools
         ↓
   ipc.ts                     ← Host-side: handles IPC requests, queries DB, writes responses
```

## Implementation

### Step 1: Database Schema (src/db.ts)

Add `memory_enabled` column to `registered_groups` table:

```typescript
// In createSchema(), after existing migrations:
try {
  database.exec(
    `ALTER TABLE registered_groups ADD COLUMN memory_enabled INTEGER DEFAULT 0`,
  );
} catch { /* column already exists */ }
```

Create `memory_chunks` table and FTS virtual table:

```sql
CREATE TABLE IF NOT EXISTS memory_chunks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  group_folder TEXT NOT NULL,
  source_file TEXT NOT NULL,
  chunk_index INTEGER NOT NULL,
  content TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now')),
  UNIQUE(source_file, chunk_index)
);
CREATE INDEX IF NOT EXISTS idx_memory_chunks_group ON memory_chunks(group_folder);
```

**IMPORTANT:** Use `tokenize='trigram'` for the FTS5 table to support Chinese substring matching. The default tokenizer splits on whitespace and cannot match Chinese characters:

```sql
CREATE VIRTUAL TABLE memory_chunks_fts USING fts5(
  content,
  content='memory_chunks',
  content_rowid='id',
  tokenize='trigram'
);
```

Add triggers to keep FTS in sync:

```sql
CREATE TRIGGER memory_chunks_ai AFTER INSERT ON memory_chunks BEGIN
  INSERT INTO memory_chunks_fts(rowid, content) VALUES (new.id, new.content);
END;
CREATE TRIGGER memory_chunks_ad AFTER DELETE ON memory_chunks BEGIN
  INSERT INTO memory_chunks_fts(memory_chunks_fts, rowid, content) VALUES('delete', old.id, old.content);
END;
CREATE TRIGGER memory_chunks_au AFTER UPDATE ON memory_chunks BEGIN
  INSERT INTO memory_chunks_fts(memory_chunks_fts, rowid, content) VALUES('delete', old.id, old.content);
  INSERT INTO memory_chunks_fts(rowid, content) VALUES (new.id, new.content);
END;
```

Add FTS migration in `initDatabase()` — detect old tokenizer and rebuild:

```typescript
// Migrate FTS table to trigram tokenizer if needed
try {
  const row = db.prepare("SELECT sql FROM sqlite_master WHERE name = 'memory_chunks_fts'").get();
  if (row && !row.sql.includes('trigram')) {
    db.exec('DROP TABLE IF EXISTS memory_chunks_fts');
    db.exec('DROP TRIGGER IF EXISTS memory_chunks_ai');
    db.exec('DROP TRIGGER IF EXISTS memory_chunks_ad');
    db.exec('DROP TRIGGER IF EXISTS memory_chunks_au');
  }
} catch { /* table doesn't exist yet */ }

createSchema(db);

// Rebuild FTS index from existing chunks
try {
  const count = db.prepare('SELECT COUNT(*) as c FROM memory_chunks').get().c;
  const ftsCount = db.prepare("SELECT COUNT(*) as c FROM memory_chunks_fts").get().c;
  if (count > 0 && ftsCount === 0) {
    db.exec("INSERT INTO memory_chunks_fts(memory_chunks_fts) VALUES('rebuild')");
  }
} catch { /* ignore */ }
```

### Step 2: DB Query Functions (src/db.ts)

**IMPORTANT:** Use `LIKE` instead of FTS5 `MATCH` for search queries. Trigram tokenizer requires minimum 3 characters, and `LIKE` is more reliable for mixed Chinese/English content on small datasets:

```typescript
export function upsertMemoryChunk(chunk: {
  group_folder: string;
  source_file: string;
  chunk_index: number;
  content: string;
}): void {
  db.prepare(
    `INSERT INTO memory_chunks (group_folder, source_file, chunk_index, content)
     VALUES (?, ?, ?, ?)
     ON CONFLICT(source_file, chunk_index) DO UPDATE SET content = excluded.content, created_at = datetime('now')`,
  ).run(chunk.group_folder, chunk.source_file, chunk.chunk_index, chunk.content);
}

export function searchMemory(query: string, groupFolder: string, limit = 5): MemorySearchResult[] {
  return db.prepare(
    `SELECT group_folder, source_file, chunk_index, content, 0 as rank
     FROM memory_chunks
     WHERE content LIKE ? AND group_folder = ?
     ORDER BY created_at DESC
     LIMIT ?`,
  ).all(`%${query}%`, groupFolder, limit);
}

export function searchMemoryAllGroups(query: string, limit = 5): MemorySearchResult[] {
  return db.prepare(
    `SELECT group_folder, source_file, chunk_index, content, 0 as rank
     FROM memory_chunks
     WHERE content LIKE ?
     ORDER BY created_at DESC
     LIMIT ?`,
  ).all(`%${query}%`, limit);
}

export function deleteMemoryChunksByGroup(groupFolder: string): void {
  db.prepare('DELETE FROM memory_chunks WHERE group_folder = ?').run(groupFolder);
}

export function deleteMemoryChunksByFile(sourceFile: string): void {
  db.prepare('DELETE FROM memory_chunks WHERE source_file = ?').run(sourceFile);
}

export function setMemoryEnabled(folder: string, enabled: boolean): void {
  db.prepare('UPDATE registered_groups SET memory_enabled = ? WHERE folder = ?')
    .run(enabled ? 1 : 0, folder);
}

export function getMemoryEnabledGroups(): string[] {
  return db.prepare('SELECT folder FROM registered_groups WHERE memory_enabled = 1')
    .all().map((r: any) => r.folder);
}
```

### Step 3: Memory Indexer (src/memory-indexer.ts)

Create a new file that periodically scans `memory/` and `conversations/` directories for enabled groups and indexes `.md` files into the database.

Key behaviors:
- **Runs immediately on startup**, then schedules based on activity
- Adaptive interval: 30s when group is active (messages in last 5min), 30min for moderate, 1h idle, 5min when nothing enabled
- Chunks text at ~1600 chars on paragraph boundaries
- Tracks file modification times to skip unchanged files
- Deletes old chunks before re-indexing a file (handles edits/shrinks)

```typescript
export function startMemoryIndexer(): void {
  logger.info('Memory indexer started');
  try {
    indexEnabledGroups();  // Run immediately
  } catch (err) {
    logger.error({ err }, 'Memory indexer initial run error');
  }
  scheduleNext();
}
```

Also export `cleanupGroupMemory()` which deletes DB chunks + filesystem `memory/` directory.

### Step 4: MCP Tools (container/agent-runner/src/ipc-mcp-stdio.ts)

Add three IPC-based MCP tools available to agents inside containers:

**memory_search** (all groups):
- `query: string` — search keywords
- `limit?: number` — max results (default 5)
- `targetGroup?: string` — main-only: specify group folder or `"*"` for all groups

**memory_enable** (main group only):
- `folder: string` — group folder name
- `enabled: boolean` — true to enable, false to disable + cleanup

**memory_cleanup** (main group only):
- `folder: string` — group folder name

These tools write IPC task files and poll for response files from the host.

### Step 5: IPC Handler (src/ipc.ts)

Handle the three memory task types in the IPC task processor:

- `memory_search`: calls `searchMemory()` or `searchMemoryAllGroups()`, writes JSON results to response file
- `memory_enable`: authorization check (main only), calls `setMemoryEnabled()` + `cleanupGroupMemory()` on disable
- `memory_cleanup`: authorization check (main only), calls `cleanupGroupMemory()`

### Step 6: Auto-enable Main Group (src/index.ts)

After `startMemoryIndexer()`, auto-enable memory for the main group:

```typescript
startMemoryIndexer();

if (!getMemoryEnabledGroups().includes(MAIN_GROUP_FOLDER)) {
  setMemoryEnabled(MAIN_GROUP_FOLDER, true);
  logger.info('Auto-enabled memory for main group');
}
```

### Step 7: Pre-Compact Hook (container/agent-runner/src/index.ts)

The existing `createPreCompactHook` archives transcripts to `conversations/` and extracts `[REMEMBER]`/`[FACT]` tagged entries to `memory/daily/YYYY-MM-DD.md`. This is a secondary mechanism — the primary way agents create memories is by writing directly to `memory/` files.

### Step 8: Update CLAUDE.md (groups/global/CLAUDE.md)

Tell agents to write memory files directly instead of relying on `[REMEMBER]` tags:

```markdown
## Memory

The `memory/` folder stores structured memories that persist across sessions:
- `memory/facts.md` — permanent facts (preferences, contacts, decisions)
- `memory/daily/YYYY-MM-DD.md` — daily event logs

When you learn something worth remembering, *write it directly* to the memory files:
- Append to `memory/facts.md` for long-term facts
- Append to `memory/daily/YYYY-MM-DD.md` for time-specific events

Do this proactively — don't wait to be asked.

Use `memory_search` to recall information from past sessions.
```

---

## Files Modified

| File | Changes |
|------|---------|
| `src/db.ts` | `memory_chunks` table, FTS5 with trigram, LIKE-based search functions, `memory_enabled` column, migration logic |
| `src/memory-indexer.ts` | **New file.** Periodic indexer with adaptive intervals, text chunking, cleanup |
| `src/index.ts` | Import memory functions, call `startMemoryIndexer()`, auto-enable main group |
| `src/ipc.ts` | Handle `memory_search`, `memory_enable`, `memory_cleanup` IPC tasks |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | `memory_search`, `memory_enable`, `memory_cleanup` MCP tools |
| `container/agent-runner/src/index.ts` | Pre-compact hook: archive transcripts + extract `[REMEMBER]` tags |
| `groups/global/CLAUDE.md` | Memory section: direct file writes instead of `[REMEMBER]` tags |

---

## Key Design Decisions

1. **LIKE over FTS5 MATCH** — FTS5 default tokenizer can't handle Chinese; trigram requires 3+ chars. LIKE works for any language on small memory datasets.
2. **Direct file writes over [REMEMBER] tags** — `[REMEMBER]` only triggers on compact (long conversations). Direct writes give immediate persistence.
3. **Main group controls all** — Only main group can enable/disable/cleanup other groups' memory. Main group auto-enables on startup.
4. **Adaptive indexing interval** — Saves resources when idle, responsive when active.
5. **IPC-based tools** — Agents run in containers, so memory tools use IPC files to communicate with the host DB.

---

## Verification

```bash
# 1. Build and restart
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# 2. Check logs
grep -i memory logs/nanoclaw-stdout.log | tail -10
# Should see: Memory indexer started, Auto-enabled memory for main group

# 3. Create test data
mkdir -p groups/main/memory
echo "- User prefers morning reminders at 8am" > groups/main/memory/facts.md

# 4. Wait ~30s for indexer, then verify
node -e "
const { initDatabase, searchMemory } = await import('./dist/db.js');
initDatabase();
console.log(searchMemory('reminders', 'main'));
"

# 5. Test in chat: ask agent "你记得关于提醒的事吗"
```

---

## Removing Memory System

1. Delete `src/memory-indexer.ts`
2. Remove memory-related functions and schema from `src/db.ts`
3. Remove memory IPC handlers from `src/ipc.ts`
4. Remove memory MCP tools from `container/agent-runner/src/ipc-mcp-stdio.ts`
5. Remove memory section from `groups/global/CLAUDE.md`
6. Revert `src/index.ts` changes (remove `startMemoryIndexer`, auto-enable)
7. Drop tables manually if needed:
   ```sql
   DROP TABLE IF EXISTS memory_chunks_fts;
   DROP TABLE IF EXISTS memory_chunks;
   ```
8. Rebuild: `npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw`
