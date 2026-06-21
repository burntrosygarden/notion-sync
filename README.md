# notion-sync

Bidirectional sync between local Markdown files and Notion pages. Designed as an agent skill so that AI assistants (QoderWork, Claude Code, Codex, etc.) automatically keep your local docs and Notion pages in sync.

## Why

If you write locally in Markdown and sync to Notion, the two versions diverge quickly. You edit on Notion during review, then next time you ask AI to work on the local file — it doesn't know Notion has a newer version and overwrites your changes. Or the other way around.

`notion-sync` solves this with a lightweight manifest file (`sync-manifest.json`) that tracks which local files map to which Notion pages, and what the current sync state is. The AI reads this manifest before every edit, pulls the latest version if needed, and pushes back after editing. Think of it like git status, but for local ↔ Notion.

## How It Works

### Three-layer architecture

```
┌─────────────────────────────────────────────────┐
│  AGENTS.md          (always loaded into context)│
│  → "you MUST check sync-manifest.json first"   │
├─────────────────────────────────────────────────┤
│  notion-sync skill  (how to do it)              │
│  → pull, push, status, conflict resolution      │
├─────────────────────────────────────────────────┤
│  HEARTBEAT.md       (periodic patrol)           │
│  → flag stale / conflicting entries             │
└─────────────────────────────────────────────────┘
```

- **AGENTS.md** is the trigger. It's a project-level convention file that AI agents read at the start of every session. The Notion Sync section tells the agent: before editing any `.md` file, check the manifest.
- **The skill** (`SKILL.md`) contains the actual procedures — how to pull from Notion, push to Notion, compute hashes, detect changes, and resolve conflicts.
- **HEARTBEAT.md** is the safety net. It runs periodic checks to catch files that slipped through.

### Sync states

Each file in the manifest has a status:

| Status | Meaning |
|--------|---------|
| `synced` | Local and Notion are identical (hashes match) |
| `local-ahead` | Local file changed since last sync; Notion is up to date |
| `remote-ahead` | Notion page changed since last sync; local is stale |
| `conflict` | Both sides changed since last sync — needs human decision |
| `pending-init` | Registered but hashes not yet computed (first-time setup) |

### The editing workflow

```
User: "help me edit paper-draft.md"
  │
  ▼
AI reads sync-manifest.json
  │
  ├── file not in manifest → edit directly, no sync
  │
  ├── status: pending-init → run Init (compute hashes)
  │
  ├── status: synced → edit directly
  │
  ├── status: local-ahead → edit directly
  │
  ├── status: remote-ahead → Pull from Notion first, then edit
  │
  └── status: conflict → show diff, ask user which version to keep
  │
  ▼
AI edits the local .md file
  │
  ▼
AI pushes to Notion, updates manifest → synced
```

## Project Setup

### 1. Create `sync-manifest.json`

Place it in your project root. See `sync-manifest.example.json` for the full schema. Each entry maps a local file to a Notion page:

```json
{
  "project": "my-research",
  "entries": [
    {
      "local_path": "drafts/paper.md",
      "notion_page_id": "a1b2c3d4e5f678901234567890abcdef",
      "title": "My Paper",
      "status": "pending-init",
      "last_synced": null,
      "local_hash": null,
      "notion_hash": null
    }
  ]
}
```

The `notion_page_id` is the 32-character UUID from the Notion page URL. For example, in `https://notion.so/My-Page-a1b2c3d4e5f678901234567890abcdef`, the ID is `a1b2c3d4e5f678901234567890abcdef` (dashes removed).

Start with `"status": "pending-init"` and null hashes. The skill will auto-initialize on first use.

### 2. Add to your `AGENTS.md`

Append this section to your project's `AGENTS.md`:

```markdown
## Notion Sync

This project uses `sync-manifest.json` to manage bidirectional sync between local .md files and Notion pages.
- Before editing any .md file, read `sync-manifest.json` (if it exists)
- If the target file is in the manifest with status `remote-ahead` or `conflict`, sync first
- After editing, push changes to Notion and update the manifest
- Follow the full workflow defined in the `notion-sync` skill
```

### 3. (Optional) Add to `HEARTBEAT.md`

For periodic patrol, add this to your project's `HEARTBEAT.md`:

```markdown
## Notion Sync Patrol

Check sync-manifest.json in this project:
- List all entries with non-synced status
- Flag conflicts as urgent
- Warn about files unsynced for more than 7 days
```

## A Complete Example

Say you have a paper draft that exists both locally and on Notion.

**Step 1: Register the file**

Add an entry to `sync-manifest.json`:

```json
{
  "local_path": "writing/paper-draft.md",
  "notion_page_id": "38584ccbacee814f873ef0a29e0f32a8",
  "title": "Paper Draft v3",
  "status": "pending-init",
  "last_synced": null,
  "local_hash": null,
  "notion_hash": null
}
```

**Step 2: First edit**

You ask the AI to edit `paper-draft.md`. The AI sees `pending-init`, runs initialization:

1. Computes SHA-256 hash of the local file
2. Fetches the Notion page content, computes its hash
3. Stores both hashes in the manifest
4. If hashes match → `synced`. If not → `conflict` (asks you which to keep).

**Step 3: Normal editing cycle**

Later, you edit the paper on Notion. Next time you ask the AI to edit locally:

1. AI checks status → `remote-ahead` (Notion hash changed)
2. AI pulls the Notion version, overwrites local file
3. AI makes your requested edits
4. AI pushes back to Notion
5. Manifest updated → `synced`

**Step 4: Conflict**

You edit locally AND on Notion before syncing. The AI detects `conflict`:

1. AI shows you the key differences between both versions
2. You choose: keep local, keep Notion, or ask AI to merge
3. AI resolves and pushes the chosen version

## Requirements

- **Notion MCP integration** — the AI needs API access to Notion pages (via Notion MCP server or equivalent)
- **`shasum`** — for hash computation (pre-installed on macOS and Linux)
- **An AI agent that reads AGENTS.md** — QoderWork, Claude Code, Codex, or similar

## File Structure

```
your-project/
├── AGENTS.md                 ← includes Notion Sync convention
├── HEARTBEAT.md              ← optional patrol tasks
├── sync-manifest.json        ← the sync manifest
└── writing/
    └── paper-draft.md        ← your document
```

## License

MIT
