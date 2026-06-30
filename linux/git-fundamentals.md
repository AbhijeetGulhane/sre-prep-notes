# Git Fundamentals — Day 1 Deep Dive

**Date:** Week 3, Day 1
**Context:** learned by scaffolding the `search-api` repo, including three real errors hit and fixed live.

---

## The mental model

```
.git/objects/   = content-addressed database (key = SHA-1 of content)
  blob          = one file's content snapshot
  tree          = one directory snapshot (list of blob/tree hashes + names)
  commit        = points at root tree + parent commit + author/message

.git/refs/heads/<branchname>  = literally just a text file containing
                                 ONE commit hash. That's the entire
                                 definition of a branch.

HEAD            = points at the current branch (or directly at a commit
                   if "detached")
```

`git add` hashes file content immediately and writes blob objects to `.git/objects/` — it doesn't wait until commit time. This is why `git add` can be slow on large files. `git commit` then builds tree objects (one per directory) and one commit object, and moves the current branch's ref forward to point at it.

Two files with identical content are stored as exactly one object, regardless of name or path — this is what content-addressing means in practice.

## Clean setup sequence for any new repo

```bash
# git identity — set ONCE per machine, do this FIRST on any new machine
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global init.defaultBranch main

git init
git add .
git commit -m "chore: scaffold repo"

# confirm branch name BEFORE pushing
git branch

git remote add origin https://github.com/USERNAME/REPO.git
git push -u origin main
```

## Diagnostic commands worth keeping handy

```bash
git log --oneline          # confirms a commit actually exists
git status                 # what's staged vs not
git status --short         # compact — catches .gitignore eating everything
git branch -vv             # confirms tracking relationship to origin/main
git remote -v              # confirms which URL origin actually points to
git cat-file -t <hash>     # what TYPE of object is this (blob/tree/commit)?
git cat-file -p <hash>     # print an object's actual content
```

## Real errors hit today, and the actual fix

| Error | Real cause | Fix |
|---|---|---|
| `Fatal error in launcher: ...python36-32...` | Stale Windows pip launcher pointing at an old Python 3.6 install | Use `python -m pip install ...` instead of bare `pip install ...` |
| `error: src refspec main does not match any` | No commit existed yet — branches don't exist until there's a commit to point at | `git add .` then `git commit` BEFORE trying to push |
| Branch showed `master` instead of `main` after committing | Local git's default branch name was `master` (set before `init.defaultBranch main` was configured) | `git branch -m master main` |
| `Permission ... denied to <wrong-account>` / `403` | Wrong GitHub account cached in Windows Credential Manager | Clear cached credential, push again, log in as correct account |

**Habit worth keeping:** run `git branch` right after the first commit, before the first push. Costs two seconds, would have caught the master/main mismatch immediately.

## Open question — folder placement

This file currently sits in `linux/` alongside process-lifecycle.md and signals.md, but git isn't really a Linux-specific topic. Worth deciding: keep a flat structure for now, or add a dedicated `git/` folder once there's enough git-specific content to justify it (we're planning ~13 more git deep-dives across Week 3-4, so it'll likely warrant its own folder soon).
