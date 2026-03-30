# claude-workspace-sync

A Claude Code plugin that exports your project's Claude workspace to a portable JSON bundle and restores it on any machine — including automatic path reconstruction and a Claude-generated session handoff note.

## What it bundles

| Item | Scope | Details |
|------|-------|---------|
| `.claude/` | project | Agents, skills, shell tools |
| `CLAUDE.md` | project | Project context for Claude |
| `~/.claude/projects/<path>/memory/` | home | Persistent memory files |
| `.git/info/exclude` entries | project | Local gitignore rules |
| Handoff summary | generated | What was worked on, open threads, decisions |

## Install

```bash
claude plugin install https://github.com/ELIXIR-NO/claude-workspace-sync
```

## Usage

### Export (on source machine)

Run inside your project directory:

```
/workspace-export
```

Optionally specify an output filename:
```
/workspace-export my-project-bundle.json
```

This writes `workspace-export-YYYY-MM-DD.json` to the project root.

### Import (on destination machine)

1. Clone your project repo on the new machine
2. Copy the bundle JSON over (scp, airdrop, USB, etc.)
3. Open Claude Code in the project directory — fresh session or `claude --resume <uuid>`
4. Run:

```
/workspace-import workspace-export-2026-03-30.json
```

Claude will:
1. Show the handoff summary immediately
2. Reconstruct all workspace files at the correct paths
3. Patch any hardcoded absolute paths in file contents
4. Re-apply git exclude entries

Memory files are written to `~/.claude/projects/<new-encoded-path>/memory/` and auto-loaded by every future session in this project — no flags needed.

## How path reconstruction works

Paths are stored in two scopes:

- **`project`** — relative to the project root, written to `<new project root>/<path>` on import
- **`home`** — relative to `$HOME` with the encoded project path segment auto-replaced

The encoded path is derived by replacing `/` with `-` in the absolute path:
- Source: `/Users/yasin/dev/FEGA-Norway` → `-Users-yasin-dev-FEGA-Norway`
- Destination: `/Users/yamir5575/work/FEGA-Norway` → `-Users-yamir5575-work-FEGA-Norway`

Any occurrence of the old absolute paths inside file **contents** is also patched on import.

## Bundle format

```json
{
  "version": "1.0",
  "exported_at": "2026-03-30T11:41:00Z",
  "source": {
    "home": "/Users/yasin",
    "project_root": "/Users/yasin/dev/FEGA-Norway",
    "encoded_project_path": "-Users-yasin-dev-FEGA-Norway"
  },
  "files": [
    { "scope": "project", "path": ".claude/agents/service-builder.md", "content": "..." },
    { "scope": "home", "path": ".claude/projects/-Users-yasin-dev-FEGA-Norway/memory/MEMORY.md", "content": "..." }
  ],
  "git_exclude_entries": [".claude/", "CLAUDE.md", "sensitive-data-archive/"],
  "handoff": {
    "summary": "Working on FEGA-Norway TLS investigation...",
    "in_progress": ["Upstream SDA PR for ROOT_CERT_PATH split"],
    "decisions_made": ["BROKER_VALIDATE=false is a bug, not intentional"],
    "watch_out_for": ["sensitive-data-archive/ needs to be re-cloned manually"]
  }
}
```

## Notes

- The bundle JSON may contain file contents that were gitignored for a reason — do not commit it to a public repo.
- Reference repos (e.g. `sensitive-data-archive/`) are not bundled — re-clone them manually on the new machine.
- Conversation history lives server-side. Use `claude --resume <uuid>` on the new machine to continue the same session alongside the restored workspace.

## License

MIT
