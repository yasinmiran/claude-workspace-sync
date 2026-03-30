---
name: export
description: Export the current Claude Code workspace to a portable JSON bundle. Use when the user runs /workspace-export, asks to export their workspace, wants to move their Claude setup to another machine, or wants to back up their project's Claude context and memory.
argument-hint: [output-filename.json]
allowed-tools: [Read, Write, Glob, Bash]
---

# Workspace Export

Export the current project's Claude workspace state to a single portable JSON file.

## Step 1: Detect environment

Run:
```bash
echo "HOME=$HOME"
pwd
ENCODED=$(python3 -c "import sys; p=sys.argv[1]; print('-'+p.lstrip('/').replace('/', '-') if p.startswith('/') else p.replace('/', '-'))" "$(pwd)")
echo "ENCODED=$ENCODED"
```

The `ENCODED` value is the encoded project path used to locate memory files.
Example: `/Users/yasin/dev/FEGA-Norway` → `-Users-yasin-dev-FEGA-Norway`

## Step 2: Collect project-scoped files

Files relative to the project root (current `pwd`). For each, record scope as `"project"` and path relative to project root.

Use Glob to find all files under `.claude/` (pattern: `.claude/**/*`), skip directories.
Also check for `CLAUDE.md` and `.claude.local.md` at the project root — include if they exist.

Record each as:
```json
{ "scope": "project", "path": "<relative-path>", "content": "<full file content>" }
```

## Step 3: Collect home-scoped files

Memory files live at `~/.claude/projects/<ENCODED>/memory/`. Use Glob on that path, skip directories.

For each file, record the path **relative to `$HOME`** (strip the `$HOME/` prefix):
```json
{ "scope": "home", "path": ".claude/projects/<ENCODED>/memory/<filename>", "content": "<full file content>" }
```

If no memory directory exists yet, skip this step silently.

## Step 4: Collect git exclude entries

Read `.git/info/exclude`. Extract all non-empty, non-comment lines that appear after the last default boilerplate comment line (`# *~`). Store as an array of strings.

If the file does not exist or has no such lines, use an empty array `[]`.

## Step 5: Generate handoff summary

Based on the current conversation, write a concise handoff note:
- **summary**: 2–3 sentences describing what was worked on in this session
- **in_progress**: list of tasks or investigations not yet complete
- **decisions_made**: key technical or architectural decisions made
- **watch_out_for**: gotchas, blockers, or context that would be non-obvious in a fresh session

## Step 6: Assemble and write JSON

Get the current timestamp:
```bash
date -u +%Y-%m-%dT%H:%M:%SZ
```

Construct the bundle:
```json
{
  "version": "1.0",
  "exported_at": "<ISO 8601 timestamp>",
  "source": {
    "home": "<$HOME>",
    "project_root": "<pwd output>",
    "encoded_project_path": "<ENCODED value>"
  },
  "files": [ <all collected file objects from Steps 2 and 3> ],
  "git_exclude_entries": [ <lines from Step 4> ],
  "handoff": {
    "summary": "<paragraph>",
    "in_progress": ["<item>"],
    "decisions_made": ["<decision>"],
    "watch_out_for": ["<gotcha>"]
  }
}
```

Output filename: use the argument if provided. Otherwise: `workspace-export-<YYYY-MM-DD>.json` (derive date from the timestamp).

Write the JSON file to the project root.

## Step 7: Confirm to user

Report:
- Full path of the written file
- Number of project-scoped files bundled
- Number of home-scoped (memory) files bundled
- File size in KB
- The full handoff summary
- Next step: copy the JSON to the new machine, open Claude Code in the project directory, and run `/workspace-import <filename>`
- Reminder: do not commit this file to a public repo as it may contain sensitive content
