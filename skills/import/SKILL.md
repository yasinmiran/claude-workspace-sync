---
name: import
description: Import a Claude Code workspace bundle JSON file. Use when the user runs /workspace-import, wants to restore a workspace from a bundle, or is setting up Claude on a new machine from an exported workspace file.
argument-hint: <bundle-file.json>
allowed-tools: [Read, Write, Bash]
---

# Workspace Import

Restore a Claude Code workspace from a portable JSON bundle onto this machine.

## Step 1: Read and validate the bundle

Read the JSON file provided as the argument. Verify it contains:
- `version` field
- `source` object with `home`, `project_root`, `encoded_project_path`
- `files` array
- `handoff` object

If any of these are missing, stop and tell the user the bundle appears invalid or corrupted.

## Step 2: Surface the handoff note immediately

Before writing any files, display the handoff to the user:

```
=== Handoff from exported workspace ===
<handoff.summary>

In progress:
- <each item in handoff.in_progress>

Decisions made:
- <each item in handoff.decisions_made>

Watch out for:
- <each item in handoff.watch_out_for>
=======================================
```

## Step 3: Detect current environment

Run:
```bash
echo "HOME=$HOME"
pwd
NEW_ENCODED=$(python3 -c "import sys; p=sys.argv[1]; print('-'+p.lstrip('/').replace('/', '-') if p.startswith('/') else p.replace('/', '-'))" "$(pwd)")
echo "NEW_ENCODED=$NEW_ENCODED"
```

This gives you the new machine's home directory, project root, and encoded project path.

## Step 4: Write each file

Process every entry in `bundle.files`:

### `scope: "project"` files

- Target path: `<current pwd>/<file.path>`
- Before writing, replace all occurrences of `bundle.source.project_root` in `file.content` with the current `pwd`
- Create any missing parent directories using Bash: `mkdir -p <parent-dir>`
- Write the file using the Write tool

### `scope: "home"` files

- Take `file.path` (which contains `bundle.source.encoded_project_path`)
- Replace `bundle.source.encoded_project_path` with `NEW_ENCODED` to get the reconstructed path
- Target path: `<$HOME>/<reconstructed-path>`
- Before writing, replace in `file.content`:
  - All occurrences of `bundle.source.project_root` → current `pwd`
  - All occurrences of `bundle.source.home` → current `$HOME`
- Create any missing parent directories using Bash: `mkdir -p <parent-dir>`
- Write the file using the Write tool

## Step 5: Apply git exclude entries

Read `.git/info/exclude` (create it if absent with `mkdir -p .git/info && touch .git/info/exclude`).

For each entry in `bundle.git_exclude_entries`:
- Check if the exact line already exists in the file
- If not present, append it

Do not duplicate entries that are already there.

## Step 6: Confirm to user

Report:
- Every file written with its final absolute path
- Number of content patches applied (files where path strings were replaced)
- Git exclude entries added vs skipped (already present)
- Reminder: any reference repos mentioned in memory files (e.g. `sensitive-data-archive/`) will need to be re-cloned manually
