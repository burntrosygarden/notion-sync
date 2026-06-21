# notion-sync

Bidirectional sync between local Markdown files and Notion pages, designed as an agent skill for AI-assisted writing workflows.

## Problem

When you write locally in Markdown and sync to Notion, the two versions diverge quickly. AI tools editing your files have no way to know which version is current, leading to overwrites and lost changes.

## Solution

A three-layer architecture:

- **`sync-manifest.json`** — lives in each project root, maps local `.md` files to Notion page IDs, tracks sync status with git-like states (`synced`, `local-ahead`, `remote-ahead`, `conflict`, `pending-init`)
- **`AGENTS.md` convention** — a project-level instruction that tells AI agents to check the manifest before every edit and push after
- **`HEARTBEAT.md` patrol** — periodic check that flags stale or conflicting entries

## Workflow

1. **Before editing**: AI reads `sync-manifest.json`, pulls the latest Notion version if the remote is ahead
2. **During editing**: AI works on the local `.md` file
3. **After editing**: AI pushes changes back to Notion and updates the manifest

Conflicts (both sides changed) are surfaced to the user for resolution.

## Usage

This is an agent skill — it's meant to be installed into an AI assistant's skill directory (e.g. QoderWork, Claude Code). Once installed, the AI follows the sync protocol automatically when working in any project that has a `sync-manifest.json`.

### Project setup

1. Create a `sync-manifest.json` in your project root (see `sync-manifest.example.json` for the schema)
2. Add the Notion Sync convention to your project's `AGENTS.md` (see SKILL.md § User Integration)
3. Optionally add the patrol task to your `HEARTBEAT.md`

### Registering files

Each entry in the manifest maps a local path to a Notion page ID:

```json
{
  "local_path": "drafts/paper.md",
  "notion_page_id": "a1b2c3d4e5f678901234567890abcdef",
  "title": "My Paper",
  "status": "synced",
  "last_synced": "2026-06-21T10:00:00+08:00",
  "local_hash": "sha256:abc123...",
  "notion_hash": "sha256:def456..."
}
```

## Requirements

- Notion MCP integration (for API access to pages)
- `shasum` for hash computation (available on macOS/Linux by default)

## License

MIT
