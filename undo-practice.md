# Git Toolkit

This file documents the Git undo and recovery tools I've learned.

## Reset

- git reset --soft HEAD~1: undo last commit, keep changes staged
- git reset --mixed HEAD~1: undo last commit, keep changes in working directory (default)
- git reset --hard HEAD~1: undo last commit and discard all changes (dangerous!)
- Only use reset on commits that haven't been pushed


## Revert

- git revert HEAD: create a new commit that undoes the last commit
- git revert is safe for shared/pushed branches because it doesn't rewrite history
- The original commit stays in the log, plus a new "undo" commit is added
- WRONG: always rebase shared branches to keep history clean


## Reflog

- git reflog: shows everywhere HEAD has pointed (commits, resets, checkouts)
- Reflog entries last about 90 days before being garbage collected
- To recover: find the SHA in reflog, then git branch <name> <SHA>


## Cherry-pick

- git cherry-pick <SHA>: apply a specific commit to the current branch
- Creates a new commit with the same changes but a different SHA
- Use for hotfixes: fix on feature branch, cherry-pick to main
