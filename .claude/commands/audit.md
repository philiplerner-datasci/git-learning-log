---
description: Storage audit — scan given folders (or the whole repo) for duplicate files, large+stale files, and empty folders; report and save to reports/storage-audit.md
argument-hint: [folder1] [folder2] ... [--size-threshold-mb N] [--stale-months N]
---

Run a storage audit and report the results — never delete, move, or modify
any scanned file while doing this. This command is meant to be run
repeatedly (e.g. monthly), so repeat it the same way each time results are
compared across runs.

## Parse $ARGUMENTS

- Any non-flag tokens are folder paths, relative to the repo root, to scan
  (recursively). If none are given, scan the whole repo root, excluding
  `.git/` and common vendor/build directories (`node_modules/`, `venv/`,
  `.venv/`, `__pycache__/`, `dist/`, `build/`, `.next/`, `target/` — skip
  any of these that exist).
- `--size-threshold-mb N` — size threshold for "large" files. Default **100**.
- `--stale-months N` — staleness threshold since last modified. Default **6**.

## 1. Scan

For each file under the scanned folder(s), resolve its effective size and
last-modified date:

- If the file's containing folder has a `manifest.json` with an `entries`
  array, and one entry's `path` matches this file's path relative to that
  folder, use that entry's `mock_size_bytes` / `mock_last_modified` instead
  of the real file — this supports repos (like mock/fixture setups) where
  on-disk placeholders stand in for larger real files. This will not apply
  in most repos; that's expected.
- Otherwise, use the file's real on-disk size and modified time.
- Skip `manifest.json` itself and `.gitkeep` placeholder files from the
  file listing.

Write a short Python script to do this scan (don't hand-wave the numbers —
compute and print them so they can be verified) covering:

1. **Duplicates** — group files by `(basename, size)` across all scanned
   folders combined; any group with more than one member is a duplicate.
   Recoverable space = size × (count − 1) per group, keeping one copy.
2. **Large + stale** — files whose size exceeds the size threshold *and*
   whose last-modified date is older than the staleness threshold from
   today. Recoverable space = full file size, but flag these for review
   rather than treating them as safe-to-delete.
3. **Empty folders** — any directory in scope with no files and no
   subdirectories, other than a `.gitkeep` placeholder.

Run the script and sanity-check its output before writing the report.

## 2. Report

Compose `reports/storage-audit.md` (relative to repo root), overwriting
any previous run, in this structure:

```markdown
# Storage Audit Report

**Scope:** <scanned folder(s), or "repo root" if none given>
**Date:** <today's date>
**Method:** File attributes taken from real on-disk stats, or from a
folder's `manifest.json` where present (simulated size/modified-date for
mock fixtures).

## 1. Duplicate Files

Files matched by identical name **and** size.

| File | Locations | Size (each) | Recoverable |
|---|---|---|---|
| ... one row per duplicate group, or "_None found._" if empty ...

**Subtotal: <sum>**

## 2. Large Files (><size threshold>) Not Modified in <stale threshold>+

| File | Size | Last Modified | Age |
|---|---|---|---|
| ... one row per match, or "_None found._" if empty ...

**Subtotal: <sum>** (flagged for review — not auto-recommended for deletion)

## 3. Empty Folders

| Folder |
|---|
| ... one row per empty folder, or "_None found._" if empty ...

## Space Recovery Summary

| Category | Recoverable | Risk |
|---|---|---|
| Duplicate files | <sum> | Low — safe to remove extra copies |
| Large & stale files | up to <sum> | Needs review — could be intentional backups |
| **Total potential** | **<sum>** | |

## Recommendations

Concrete, numbered recommendations tied to the actual findings above (not
generic advice). If a category is empty, say so plainly rather than
inventing a recommendation for it.

No files were deleted or modified as part of this audit — all actions
above are recommendations only.
```

## 3. Show before saving

Print the full report in chat and wait for the user to confirm (or request
changes) before writing the file — do not silently save without showing it
first.

## 4. Save, commit, and push

After confirmation, write `reports/storage-audit.md` (create the `reports/`
directory if needed), then `git add`, commit with a message like `Update
storage audit report (<date>)`, and push to the current branch, following
whatever this repo's standard push retry practice is (retry on network
errors only, exponential backoff, up to 4 attempts).
