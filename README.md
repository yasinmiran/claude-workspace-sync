# claude-workspace-sync

A Claude Code plugin that exports your project workspace to a portable JSON bundle and imports it on any machine.

## What it bundles

- `.claude/` — agents, skills, tools
- `CLAUDE.md` — project context
- `~/.claude/projects/<path>/memory/` — persistent memory files
- `.git/info/exclude` — local gitignore entries
- A Claude-generated **handoff summary** — what was being worked on, decisions made, open threads

## Install

```bash
claude plugin install https://github.com/ELIXIR-NO/claude-workspace-sync
```

## Usage

**Export** (run inside your project):
```
/workspace-export
/workspace-export my-bundle.json
```

**Import** (run inside the project on the new machine after cloning):
```
/workspace-import workspace-export-2026-03-30.json
```

## How path reconstruction works

Paths are stored in two scopes:
- `project` — relative to the project root, reconstructed at import time from `pwd`
- `home` — relative to `$HOME` with the encoded project path segment replaced to match the new machine

Any hardcoded absolute paths found inside file **contents** are also patched on import.
