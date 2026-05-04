# The Complete Git Guide

A practical, end-to-end reference for Git — from the underlying data model to the messy real-world situations that make developers panic at 11 PM on a Friday.

---

## Table of contents

1. [What Git is and why it exists](#1-what-git-is-and-why-it-exists)
2. [How Git works internally](#2-how-git-works-internally)
3. [Installation and one-time setup](#3-installation-and-one-time-setup)
4. [The three states](#4-the-three-states)
5. [Core commands](#5-core-commands)
6. [Branching and merging](#6-branching-and-merging)
7. [Remotes and GitHub](#7-remotes-and-github)
8. [Inspecting history](#8-inspecting-history)
9. [Undoing things](#9-undoing-things)
10. [Stashing](#10-stashing)
11. [Rebasing](#11-rebasing)
12. [Tags and releases](#12-tags-and-releases)
13. [`.gitignore` and tracking files](#13-gitignore-and-tracking-files)
14. [Hooks](#14-hooks)
15. [Submodules and worktrees](#15-submodules-and-worktrees)
16. [Real-world scenarios](#16-real-world-scenarios)
17. [Recovery and emergencies](#17-recovery-and-emergencies)
18. [Workflows](#18-workflows)
19. [Best practices](#19-best-practices)
20. [Cheat sheet](#20-cheat-sheet)

---

## 1. What Git is and why it exists

**Git** is a distributed version control system. Created by Linus Torvalds in 2005 to manage the Linux kernel after the team's previous tool (BitKeeper) revoked its free license. Designed for speed, integrity, and parallel non-linear workflows.

**Distributed** means every clone of a repository contains the entire history. There is no central "master copy" at the protocol level — GitHub is just one more peer that everyone happens to agree to push to.

**Versus alternatives:**
- SVN, CVS — centralized, slow, hard to branch.
- Mercurial — similar model to Git, smaller ecosystem.
- Perforce — used in game dev for binary assets, centralized.

Git won because branching is nearly free, the data model is simple, and GitHub built a community on top of it.

---

## 2. How Git works internally

Git is fundamentally a **content-addressable key-value store**. You give it content, it returns a SHA-1 hash. You give it the hash, it returns the content.

### The four object types

All four live inside `.git/objects/` as compressed files named by their hash.

| Object | What it stores | Points to |
|---|---|---|
| **Blob** | Raw file contents (no filename, no path) | nothing |
| **Tree** | A directory listing — names mapped to hashes | blobs and other trees |
| **Commit** | Snapshot metadata: tree, parent(s), author, message | one tree + parent commit(s) |
| **Tag** | Annotated tag object (name, message, signature) | a commit |

### The chain

A commit points to a tree (the project state) and to its parent commit. Walk the parent links and you have history. A merge commit has two parents. Branches are tiny files containing one commit hash. `HEAD` points to the current branch (or directly to a commit in "detached HEAD" mode).

### What `git add` and `git commit` actually do

```bash
echo "hello" > greet.txt
git add greet.txt
# 1. Reads file, computes SHA-1 of contents
# 2. Writes blob to .git/objects/ce/0136...
# 3. Updates .git/index to map greet.txt -> ce0136...

git commit -m "First"
# 4. Builds tree object listing index entries -> writes tree blob
# 5. Builds commit object {tree, parent, author, message} -> writes commit blob
# 6. Updates .git/refs/heads/main to contain new commit hash
```

### Why snapshots beat diffs

Older systems (SVN, CVS) stored deltas between versions. Git stores full snapshots, deduplicated by hash. Two identical files anywhere in history share one blob. This makes any historical operation — `checkout`, `diff`, `log` — equally fast.

### Inspect it yourself

```bash
git cat-file -t HEAD                    # type of HEAD's object (commit)
git cat-file -p HEAD                    # pretty-print HEAD commit
git ls-tree HEAD                        # show the root tree
git rev-parse HEAD                      # full hash of HEAD
cat .git/HEAD                           # ref: refs/heads/main
cat .git/refs/heads/main                # commit hash
ls .git/objects/                        # raw object database
```

---

## 3. Installation and one-time setup

### Install

| Platform | Command |
|---|---|
| macOS | `brew install git` or use Xcode Command Line Tools |
| Ubuntu/Debian | `sudo apt install git` |
| Fedora | `sudo dnf install git` |
| Windows | Download from [git-scm.com](https://git-scm.com) (includes Git Bash) |

Verify: `git --version`

### Identity

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Sensible defaults

```bash
git config --global init.defaultBranch main
git config --global pull.rebase false           # merge on pull (safer default)
git config --global core.editor "code --wait"   # VS Code as commit editor
git config --global core.autocrlf input         # macOS/Linux
git config --global core.autocrlf true          # Windows
git config --global push.default current        # push current branch only
git config --global push.autoSetupRemote true   # auto-track on first push
git config --global rerere.enabled true         # remember conflict resolutions
git config --global fetch.prune true            # auto-clean deleted remote branches
```

### Useful aliases

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm "commit -m"
git config --global alias.last "log -1 HEAD"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.unstage "reset HEAD --"
git config --global alias.amend "commit --amend --no-edit"
```

### SSH keys for GitHub

```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub                # paste into GitHub Settings -> SSH keys
ssh -T git@github.com                    # test
```

### View your config

```bash
git config --list                        # all config
git config --list --show-origin          # where each value comes from
git config user.email                    # one value
```

Config lives in three places, in priority order: `.git/config` (repo) > `~/.gitconfig` (user) > `/etc/gitconfig` (system).

---

## 4. The three states

Every file in a Git project is in one of these states:

```
working directory  ── git add ──▶  staging (index)  ── git commit ──▶  repository (.git)
     (modified)                       (staged)                            (committed)
```

- **Working directory** — actual files you edit on disk.
- **Staging area / index** — a snapshot prepared for the next commit. Lets you build commits deliberately.
- **Repository** — the permanent, content-addressed object store inside `.git/`.

### Why staging exists

Most beginners find `git add` then `git commit` redundant. The reason: staging lets you split changes across multiple commits even if you edited everything at once.

```bash
# edited 10 files, but want 2 logical commits
git add src/auth.ts src/auth.test.ts
git commit -m "Add JWT validation"
git add src/ui/
git commit -m "Update login form styling"
```

You can even stage parts of a single file:

```bash
git add -p file.txt          # interactively pick hunks
```

---

## 5. Core commands

### Creating a repository

```bash
git init                                 # new repo in current folder
git init my-project                      # new repo in my-project/
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git   # via SSH
git clone <url> custom-folder-name
git clone --depth 1 <url>                # shallow clone (history-less, faster)
```

### Status and changes

```bash
git status                               # what's modified, staged, untracked
git status -s                            # short format
git diff                                 # working dir vs staged
git diff --staged                        # staged vs last commit (same as --cached)
git diff HEAD                            # working dir vs last commit
git diff main..feature                   # compare two branches
git diff <commit1> <commit2> -- file.txt # specific file between commits
```

### Staging

```bash
git add file.txt
git add src/                             # whole folder
git add .                                # everything in current dir
git add -A                               # everything in repo (incl. deletions)
git add -u                               # only tracked files (no new files)
git add -p                               # interactive, hunk by hunk
git add -i                               # interactive menu

git restore --staged file.txt            # unstage (modern)
git reset HEAD file.txt                  # unstage (legacy, still works)
```

### Committing

```bash
git commit -m "Short message"
git commit -m "Title" -m "Longer body explaining why"
git commit                               # opens editor for full message
git commit -a -m "msg"                   # auto-stage tracked files + commit
git commit --amend                       # edit last commit (message or contents)
git commit --amend --no-edit             # add staged changes to last commit silently
git commit --allow-empty -m "trigger CI" # commit with no changes
```

### Discarding changes

```bash
git restore file.txt                     # discard working-dir changes
git restore .                            # discard ALL working-dir changes
git restore --source=HEAD~2 file.txt     # restore file from 2 commits ago
git checkout -- file.txt                 # legacy equivalent
git clean -n                             # preview untracked files to delete
git clean -fd                            # delete untracked files + folders
```

### Removing and renaming

```bash
git rm file.txt                          # delete + stage deletion
git rm --cached file.txt                 # untrack but keep on disk
git rm -r folder/                        # remove folder
git mv old.txt new.txt                   # rename + stage
```

---

## 6. Branching and merging

### Branches

```bash
git branch                               # list local branches
git branch -a                            # list all (incl. remote)
git branch -v                            # with last commit
git branch feature/login                 # create (don't switch)
git switch feature/login                 # switch (modern)
git switch -c feature/login              # create + switch
git checkout -b feature/login            # legacy: create + switch
git switch -                             # back to previous branch (like cd -)

git branch -d feature/login              # delete (safe — refuses if unmerged)
git branch -D feature/login              # force delete
git branch -m old-name new-name          # rename
git push origin --delete feature/login   # delete remote branch
```

### Merging

```bash
git switch main
git merge feature/login                  # merge feature into main
git merge --no-ff feature/login          # always create merge commit (preserves branch shape)
git merge --squash feature/login         # collapse all feature commits into one staged change
git merge --abort                        # bail out mid-conflict
```

**Fast-forward vs merge commit:** if `main` hasn't moved since `feature` branched off, Git just moves the `main` pointer forward — no merge commit. Use `--no-ff` to force a merge commit and preserve the fact that work happened on a branch.

### Resolving merge conflicts

When Git can't auto-merge, it inserts conflict markers:

```
<<<<<<< HEAD
your version
=======
their version
>>>>>>> feature/login
```

Steps:
1. Open each conflicted file (`git status` lists them).
2. Edit to keep what you want; remove all `<<<<<<<`, `=======`, `>>>>>>>` lines.
3. `git add <file>` to mark resolved.
4. `git commit` (Git pre-fills a merge message).

Tools:
```bash
git mergetool                            # opens configured merge tool
git diff --name-only --diff-filter=U     # list unresolved files
git checkout --ours file.txt             # keep our version entirely
git checkout --theirs file.txt           # keep their version entirely
```

---

## 7. Remotes and GitHub

### Managing remotes

```bash
git remote -v                            # list remotes with URLs
git remote add origin git@github.com:user/repo.git
git remote rename origin upstream
git remote remove origin
git remote set-url origin <new-url>      # change URL (e.g. HTTPS -> SSH)
```

By convention:
- `origin` — your fork or main remote
- `upstream` — the original repo you forked from

### Fetch, pull, push

```bash
git fetch                                # download remote refs (no merge)
git fetch --all --prune                  # fetch all remotes, drop deleted branches

git pull                                 # = fetch + merge
git pull --rebase                        # = fetch + rebase (linear history)

git push                                 # push current branch to its tracked remote
git push -u origin feature/login         # first push: set upstream tracking
git push --force-with-lease              # safer force push (refuses if remote moved)
git push --force                         # dangerous; overwrites remote
git push --tags                          # push all tags
git push origin --delete feature/login   # delete remote branch
```

### `--force` vs `--force-with-lease`

`--force` will overwrite remote history blindly. If a teammate pushed since you last fetched, you wipe their work. `--force-with-lease` checks that the remote ref still matches what you last fetched, and refuses if someone else has pushed. **Always use `--force-with-lease`.**

### Tracking branches

```bash
git branch -vv                           # show what each local branch tracks
git branch -u origin/main                # set upstream for current branch
git push -u origin feature               # set upstream and push
```

### GitHub-specific concepts

- **Fork** — your server-side copy of someone else's repo.
- **Pull Request (PR)** — proposes merging one branch into another, with review.
- **Issues** — bug/feature tracker.
- **Actions** — CI/CD that runs on push, PR, schedule, etc.
- **Pages** — static site hosting from a branch (e.g. `gh-pages`).
- **Releases** — tagged downloadable archives + release notes.

Typical contribution flow to someone else's project:

```bash
# 1. Fork on GitHub UI
git clone git@github.com:yourname/their-repo.git
cd their-repo
git remote add upstream git@github.com:original/their-repo.git
git switch -c fix/typo
# ...edit...
git push -u origin fix/typo
# 2. Open PR on GitHub from yourname:fix/typo -> original:main
# Keeping fork synced:
git fetch upstream
git switch main
git merge upstream/main
git push
```

---

## 8. Inspecting history

```bash
git log                                  # full log
git log --oneline                        # one line per commit
git log --oneline --graph --decorate --all   # visual graph of all branches
git log -p file.txt                      # commits + diffs for one file
git log --stat                           # commits + file change summary
git log --since="2 weeks ago"
git log --author="Alice"
git log --grep="bug"                     # search commit messages
git log -S "functionName"                # commits that added/removed a string ("pickaxe")
git log main..feature                    # commits in feature not in main
git log --follow file.txt                # follow renames

git show HEAD                            # last commit details + diff
git show <hash>                          # any commit
git show HEAD:path/to/file.txt           # file contents at HEAD

git blame file.txt                       # who last touched each line
git blame -L 10,20 file.txt              # blame for lines 10-20

git shortlog -sn                         # commit count per author
git reflog                               # log of HEAD movements (your safety net)
```

### Reading a commit hash

You can refer to commits many ways:

```
HEAD            # current commit
HEAD~1          # one commit before HEAD (parent)
HEAD~3          # three before
HEAD^           # parent (same as HEAD~1)
HEAD^2          # second parent of a merge commit
main            # tip of main branch
abc123          # short hash (≥4 chars, must be unique)
v1.0.0          # tag
@{yesterday}    # where HEAD was yesterday
```

---

## 9. Undoing things

This is where most panic happens. Memorize this section.

### "I haven't committed yet"

```bash
git restore file.txt                     # discard working-dir changes to one file
git restore .                            # discard all working-dir changes
git restore --staged file.txt            # unstage but keep changes
git checkout -- file.txt                 # legacy discard
```

### "I committed, but didn't push"

```bash
git commit --amend                       # change last commit's message or contents
git reset --soft HEAD~1                  # undo commit, keep changes staged
git reset --mixed HEAD~1                 # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1                  # undo commit, DISCARD changes (destructive)
```

The three reset modes — burn this into memory:

| Mode | HEAD | Index (stage) | Working dir |
|---|---|---|---|
| `--soft` | moved | unchanged | unchanged |
| `--mixed` (default) | moved | reset | unchanged |
| `--hard` | moved | reset | reset (data loss!) |

### "I committed and pushed, and need to undo it"

If others may have pulled your commit, **never rewrite history**. Use `git revert`:

```bash
git revert <hash>                        # creates a NEW commit that undoes <hash>
git revert HEAD                          # revert most recent commit
git revert -m 1 <merge-hash>             # revert a merge commit (keep first-parent line)
git revert --no-commit <hash1> <hash2>   # stage reverts without committing
```

If it's truly safe to rewrite (your private feature branch, no one else has it):

```bash
git reset --hard HEAD~1
git push --force-with-lease
```

### "I committed to the wrong branch"

```bash
# made commits on main that should be on feature/x:
git switch -c feature/x                  # create branch from where you are
git switch main
git reset --hard origin/main             # move main back to where remote is
```

### "I want to undo a specific old commit but keep everything since"

```bash
git revert <hash>                        # safest — adds undo commit
# or interactive rebase to drop it (rewrites history):
git rebase -i <hash>~1
# in editor, change "pick" to "drop" on that line
```

### "I want to recover a deleted branch / lost commit"

```bash
git reflog                               # find the lost commit hash
git checkout <hash>                      # see it
git switch -c recovered-branch <hash>    # create branch pointing at it
```

The reflog tracks every HEAD movement for ~90 days by default. As long as a commit was reachable from HEAD at some point, you can find it here.

### "Help, I `git reset --hard` and lost work"

```bash
git reflog                               # find the commit before the reset
git reset --hard <hash>                  # back to that state
```

If files were never committed and you ran `--hard`, they're usually gone. Lesson: commit (even WIP) before risky operations, or use `git stash`.

---

## 10. Stashing

Temporarily shelve changes without committing — useful when you need to switch contexts mid-edit.

```bash
git stash                                # stash tracked changes
git stash -u                             # include untracked files
git stash -a                             # include ignored files too
git stash push -m "WIP: refactor auth"   # named stash
git stash push file1.txt file2.txt       # stash specific files

git stash list                           # show all stashes
git stash show stash@{0}                 # summary of a stash
git stash show -p stash@{0}              # full diff

git stash pop                            # apply latest stash and remove it
git stash apply                          # apply but keep in stash list
git stash apply stash@{2}                # apply a specific one
git stash drop stash@{0}                 # delete a stash
git stash clear                          # delete all stashes

git stash branch new-branch stash@{0}    # create branch from stash
```

Stashes survive across branches but live only locally — they don't push.

---

## 11. Rebasing

Rebasing rewrites commits onto a new base. Use it to keep history linear or to clean up messy commits before sharing.

### Basic rebase

```bash
git switch feature/x
git rebase main                          # replay feature/x commits on top of main
```

Compare:
- **Merge** keeps both branches' history; creates a merge commit.
- **Rebase** rewrites your branch's commits onto the target — appears as if you branched from the latest main.

### Interactive rebase — the power tool

```bash
git rebase -i HEAD~5                     # edit last 5 commits
git rebase -i main                       # edit all commits since main
```

Opens an editor with each commit. Replace `pick` with:

| Action | Effect |
|---|---|
| `pick` | use commit as-is |
| `reword` | use commit, edit message |
| `edit` | pause to amend commit |
| `squash` | combine into previous commit, merge messages |
| `fixup` | combine into previous commit, discard this message |
| `drop` | remove commit entirely |

You can also reorder lines to reorder commits.

### Rebase conflicts

```bash
# fix conflicts in files
git add <files>
git rebase --continue
git rebase --skip                        # skip the conflicting commit
git rebase --abort                       # bail out, return to pre-rebase state
```

### The Golden Rule of Rebasing

**Never rebase commits that have been pushed and that others may have based work on.** Rebasing rewrites hashes — anyone who pulled the old hashes will have a divergent history nightmare.

Rebase your local feature branch freely. Rebase shared branches (like `main`) never.

---

## 12. Tags and releases

Tags mark specific commits — usually releases.

```bash
git tag                                  # list
git tag v1.0.0                           # lightweight tag (just a pointer)
git tag -a v1.0.0 -m "First release"     # annotated tag (recommended)
git tag -a v1.0.0 <hash>                 # tag a specific commit
git tag -s v1.0.0 -m "msg"               # GPG-signed

git show v1.0.0                          # show tag info
git push origin v1.0.0                   # push one tag
git push --tags                          # push all tags
git tag -d v1.0.0                        # delete locally
git push origin --delete v1.0.0          # delete remote tag

git checkout v1.0.0                      # check out at a tag (detached HEAD)
```

**Annotated vs lightweight:** annotated tags are full Git objects with their own metadata (tagger, date, message). Always use annotated for releases.

GitHub turns tags pushed in `vX.Y.Z` form into Releases automatically when you create one through the UI or `gh release create v1.0.0`.

---

## 13. `.gitignore` and tracking files

`.gitignore` lists patterns Git should ignore. Place at repo root (or any subfolder for scoped ignores).

```gitignore
# comments start with #

# ignore all .log files
*.log

# but track this one
!important.log

# ignore a folder
node_modules/
dist/

# ignore a file in any folder
**/secrets.env

# ignore at the root only
/build/

# ignore by extension in a folder
src/**/*.tmp
```

### "It's still being tracked!"

`.gitignore` only ignores **untracked** files. If a file was already tracked, you must remove it from the index:

```bash
git rm --cached file.txt                 # untrack but keep on disk
git rm -r --cached node_modules/         # untrack a folder
git commit -m "Stop tracking node_modules"
```

### Useful global ignore

```bash
git config --global core.excludesfile ~/.gitignore_global
```

A typical `~/.gitignore_global` ignores OS junk like `.DS_Store`, `Thumbs.db`, and editor folders like `.vscode/` and `.idea/` so they never accidentally enter any repo.

### Templates

[github.com/github/gitignore](https://github.com/github/gitignore) has battle-tested templates per language and stack. GitHub also offers them when you create a repo.

---

## 14. Hooks

Hooks are scripts in `.git/hooks/` that run on specific events. Useful for automated linting, tests, commit message validation.

| Hook | Fires when |
|---|---|
| `pre-commit` | before commit is created — run linters, tests |
| `commit-msg` | validate commit message format |
| `pre-push` | before pushing — run full test suite |
| `post-merge` | after a merge — re-install deps, notify |
| `post-checkout` | after switching branches |

Hooks live in `.git/hooks/` (not versioned). For team-shared hooks, use **Husky** (Node), **pre-commit** (Python), or **lefthook** (cross-platform) which check committed config files into the repo.

Example `pre-commit`:

```bash
#!/bin/sh
npm run lint || exit 1
npm test || exit 1
```

Make it executable: `chmod +x .git/hooks/pre-commit`.

---

## 15. Submodules and worktrees

### Submodules — repos inside repos

```bash
git submodule add https://github.com/user/lib.git vendor/lib
git submodule init                       # initialize after clone
git submodule update                     # pull submodule contents
git submodule update --init --recursive  # nested submodules
git clone --recurse-submodules <url>     # clone with submodules
git submodule update --remote            # update submodules to their latest
```

Submodules pin a parent repo to a specific commit of a child repo. Useful for shared libraries, but operationally awkward — most teams prefer package managers (npm, cargo, pip) when possible.

### Worktrees — multiple branches checked out at once

```bash
git worktree add ../project-hotfix hotfix/urgent
# now ../project-hotfix is a full working tree on hotfix/urgent
# while the original folder stays on main
git worktree list
git worktree remove ../project-hotfix
```

Lets you fix a bug on a hotfix branch without stashing your main work. Faster than cloning twice.

---

## 16. Real-world scenarios

The situations that actually come up.

### "I made changes on `main` directly. I should've branched."

```bash
git switch -c feature/proper-branch      # creates branch from current state
git switch main
git reset --hard origin/main             # main back to remote state
git switch feature/proper-branch         # continue work
```

### "My PR has 30 messy commits. Reviewer wants a clean history."

```bash
git rebase -i main                       # mark all but first as "squash" or "fixup"
# or simpler — squash on merge via the GitHub PR "Squash and merge" button
```

### "I committed a secret (API key, password)."

If unpushed: `git reset --soft HEAD~1`, remove the secret, recommit.

If pushed: **rotate the secret immediately** — assume it's compromised — *then* clean history:

```bash
# Modern tool — git-filter-repo (recommended)
pip install git-filter-repo
git filter-repo --path config/secrets.yml --invert-paths
git push --force-with-lease

# Tell collaborators to re-clone — their old clones still have the secret
```

The secret existed in a public commit even briefly. Bots scan GitHub continuously. Rotate first, clean second.

### "I need to bring one commit from another branch to mine."

```bash
git switch my-branch
git cherry-pick <hash>                   # apply that commit's diff here
git cherry-pick A..B                     # range (exclusive of A, inclusive of B)
git cherry-pick --no-commit <hash>       # stage only, don't commit
```

### "I need to update my feature branch with latest main."

Two options. Pick one and stick with it on a given branch:

```bash
# Option A: merge (preserves real history, adds merge commits)
git switch feature/x
git merge main

# Option B: rebase (linear history, rewrites feature commits)
git switch feature/x
git rebase main
git push --force-with-lease              # required after rebase
```

Convention varies by team. Many teams rebase feature branches onto main, then `--no-ff` merge the PR into main.

### "I'm investigating which commit broke something."

`git bisect` — binary search through history:

```bash
git bisect start
git bisect bad                           # current commit is broken
git bisect good v1.0.0                   # this old version worked
# git checks out a commit halfway between
# you test, then say:
git bisect good                          # or: git bisect bad
# repeat until Git names the culprit
git bisect reset                         # done
```

Even better — automate it:

```bash
git bisect start HEAD v1.0.0
git bisect run npm test                  # exits 0 = good, non-zero = bad
```

### "Two of us edited the same file and now there's a conflict on push."

```bash
git pull --rebase                        # pulls their commits, replays yours on top
# resolve conflicts file by file
git add <files>
git rebase --continue
git push
```

### "I want to see what changed in a file over time."

```bash
git log -p file.txt                      # commits with diffs
git log --follow file.txt                # follow through renames
git blame file.txt                       # last author per line
git log -S "specificString" file.txt     # commits where this string changed
```

### "I need to ignore a file I'm tracking, but only locally."

`.gitignore` doesn't help if it's tracked. Use:

```bash
git update-index --skip-worktree config/local.json
# Git pretends the file is unchanged, even if you edit it
git update-index --no-skip-worktree config/local.json   # undo
```

For never-shared local config, prefer a `local.json.example` committed file + `local.json` in `.gitignore`.

### "I want to copy a single folder from another repo, with history."

```bash
# extract folder from old repo into its own repo:
git clone old-repo extracted
cd extracted
git filter-repo --subdirectory-filter path/to/folder
# now merge into target:
cd ../new-repo
git remote add temp ../extracted
git fetch temp
git merge temp/main --allow-unrelated-histories
git remote remove temp
```

### "Our `main` branch was force-pushed and I'm out of sync."

```bash
git fetch origin
git reset --hard origin/main             # match remote state (loses local main commits!)
# if you had unique local commits, save them first:
git branch backup-main
git fetch origin
git reset --hard origin/main
# cherry-pick from backup-main as needed
```

### "I want to release a hotfix without including in-progress feature work."

Use a hotfix branch off the last release tag:

```bash
git switch -c hotfix/critical v1.2.0
# ...fix...
git tag -a v1.2.1 -m "Hotfix"
git push origin hotfix/critical v1.2.1
# then merge hotfix into main and any active release branches
```

---

## 17. Recovery and emergencies

### The reflog is your friend

```bash
git reflog                               # every HEAD movement, ~90 days
git reflog show feature/x                # reflog for a specific branch
git reset --hard HEAD@{2}                # restore HEAD to 2 movements ago
```

Anything HEAD ever pointed to is recoverable — even after `reset --hard`, deleted branches, botched rebases.

### Garbage collection

```bash
git gc                                   # cleanup loose objects
git gc --prune=now                       # aggressive cleanup (DESTROYS unreachable objects)
git fsck                                 # filesystem check — finds dangling commits
```

By default, unreachable objects survive ~2 weeks before `gc` reaps them. Don't `gc --prune=now` if you might need to recover something.

### Corrupted repository

```bash
git fsck --full                          # diagnose
# usually fixable by cloning fresh and copying uncommitted work over
```

### "I deleted my local repo, but I had pushed."

```bash
git clone <remote-url>
# everything pushed is recoverable
```

This is the killer feature of distributed VCS — the remote is a complete backup.

---

## 18. Workflows

### Trunk-based development

Everyone commits to `main` (or short-lived branches merged within a day). Requires strong CI and feature flags. Used at Google, Meta, most modern startups.

**Pros:** simple, fast feedback, no integration debt.
**Cons:** demands rigorous testing, mature CI.

### GitHub Flow

```
main (always deployable)
  └─ feature/x ──PR──▶ main
  └─ fix/y    ──PR──▶ main
```

Branch off `main`, open a PR, get reviewed, merge, deploy. Works for most web projects. Default for GitHub-native teams.

### Git Flow (heavier)

Long-lived `develop` and `main` branches, plus `feature/*`, `release/*`, `hotfix/*`.

```
main (production)
  ├─ hotfix/* ──▶ main + develop
develop (integration)
  ├─ feature/* ──▶ develop
  └─ release/* ──▶ main + develop
```

**Use when:** versioned releases (mobile apps, libraries), strict release cadence.
**Skip when:** continuous deployment to a web service.

### Forking workflow

Used in open source. Contributors fork the upstream repo, push to their fork, open PRs back. Maintainers never give write access to the main repo.

---

## 19. Best practices

### Commits

- **Atomic** — one logical change per commit. Don't mix "fix typo" and "rewrite auth" in one commit.
- **Imperative mood** — "Add login form", not "Added login form" or "Adds login form". Convention: a commit message completes the sentence "If applied, this commit will ___."
- **Subject ≤ 50 chars**, blank line, then **body wrapped at 72 chars** explaining *why*.
- **Conventional Commits** — many teams use a structured prefix:

  ```
  feat: add user profile page
  fix(auth): handle expired tokens
  docs: update README
  refactor: extract validation helper
  test: cover edge cases in date parser
  chore: bump dependencies
  ```

  Tools like `semantic-release` use these to auto-generate changelogs and version bumps.

### Branches

- **Short-lived** — merge within days, not weeks. Long branches drift and conflict.
- **Descriptive names** — `feat/user-profile`, `fix/login-redirect`, `chore/update-deps`. Avoid `my-branch`, `test`, `wip`.
- **One concern per branch** — don't pile unrelated work into the same PR.

### Pull Requests

- Keep them small (< 400 lines changed when possible). Reviewer fatigue is real.
- Self-review the diff before requesting review.
- Write a description: *what* changed, *why*, and *how to test*.
- Link related issues (`Closes #123` auto-closes on merge).

### General

- **Pull before push** to minimize divergence.
- **Never commit secrets, large binaries, or generated files.** Use `.gitignore` aggressively.
- **Sign your commits** for important repos: `git commit -S` with GPG/SSH key configured.
- **Verify before force pushing:** `git log @{u}..HEAD` shows what you're about to overwrite remotely.

---

## 20. Cheat sheet

### Setup
```bash
git config --global user.name "Name"
git config --global user.email "you@x.com"
git init | git clone <url>
```

### Daily flow
```bash
git status
git diff
git add <files> | git add -p
git commit -m "msg"
git pull --rebase
git push
```

### Branching
```bash
git switch -c feature/x
git switch main
git merge feature/x
git branch -d feature/x
```

### Inspecting
```bash
git log --oneline --graph --all
git show <hash>
git blame <file>
git reflog
```

### Undoing
```bash
git restore <file>                       # discard working changes
git restore --staged <file>              # unstage
git commit --amend                       # edit last commit
git reset --soft HEAD~1                  # undo commit, keep changes
git revert <hash>                        # safe undo of pushed commit
```

### Stash
```bash
git stash
git stash pop
git stash list
```

### Remote
```bash
git remote -v
git fetch
git push -u origin <branch>
git push --force-with-lease
```

### Rebase
```bash
git rebase main
git rebase -i HEAD~5
git rebase --continue | --abort
```

### Tags
```bash
git tag -a v1.0.0 -m "msg"
git push --tags
```

### Emergencies
```bash
git reflog                               # find lost commits
git reset --hard HEAD@{N}                # restore to N steps ago
git fsck --full                          # diagnose corruption
```

---

## Resources

- **Pro Git** (Scott Chacon, Ben Straub) — free at [git-scm.com/book](https://git-scm.com/book). The definitive reference.
- **Oh Shit, Git!?!** ([ohshitgit.com](https://ohshitgit.com)) — recovery scenarios in plain language.
- **Learn Git Branching** ([learngitbranching.js.org](https://learngitbranching.js.org)) — interactive visual tutorial.
- **Git documentation** — `git help <command>` or `man git-<command>`.
- **GitHub Docs** ([docs.github.com](https://docs.github.com)) — for GitHub-specific features.

---

*Last updated: this is a living document — fork it, improve it, send a PR.*
