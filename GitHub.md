# GitHub — Complete Notes (Basics to Advanced)
> Interview-ready reference. Every major concept, command, and workflow in one place.

---

## Table of Contents

1. [What is Git vs GitHub?](#1-what-is-git-vs-github)
2. [Core Git Concepts](#2-core-git-concepts)
3. [Setting Up Git & GitHub](#3-setting-up-git--github)
4. [Basic Git Commands](#4-basic-git-commands)
5. [Branching & Merging](#5-branching--merging)
6. [Remote Repositories](#6-remote-repositories)
7. [Pull Requests (PRs)](#7-pull-requests-prs)
8. [Merge Strategies](#8-merge-strategies)
9. [Undoing Changes](#9-undoing-changes)
10. [Stashing](#10-stashing)
11. [Tags & Releases](#11-tags--releases)
12. [Git Log & History](#12-git-log--history)
13. [Rebasing](#13-rebasing)
14. [Cherry-Picking](#14-cherry-picking)
15. [GitHub Flow vs Git Flow](#15-github-flow-vs-git-flow)
16. [Forking & Open Source Contribution](#16-forking--open-source-contribution)
17. [GitHub Actions (CI/CD)](#17-github-actions-cicd)
18. [GitHub Issues & Projects](#18-github-issues--projects)
19. [GitHub Security Features](#19-github-security-features)
20. [Advanced Git Internals](#20-advanced-git-internals)
21. [Common Interview Questions & Answers](#21-common-interview-questions--answers)

---

## 1. What is Git vs GitHub?

| | Git | GitHub |
|---|---|---|
| **Type** | Version control system (VCS) | Cloud hosting platform for Git repos |
| **Created by** | Linus Torvalds (2005) | Tom Preston-Werner et al. (2008) |
| **Works** | Locally on your machine | On the internet |
| **Without the other?** | Git works without GitHub | GitHub needs Git |

**Key point for interviews:** Git is the tool; GitHub is the service. Alternatives to GitHub include GitLab, Bitbucket, and Azure DevOps.

---

## 2. Core Git Concepts

### The Three Areas (States of a File)

```
Working Directory  →  Staging Area (Index)  →  Local Repository  →  Remote Repository
      (edit)              (git add)               (git commit)          (git push)
```

- **Working Directory** — where you edit files
- **Staging Area (Index)** — files queued for the next commit
- **Local Repository (.git folder)** — committed history stored on your machine
- **Remote Repository** — hosted on GitHub (or similar)

### File States

- **Untracked** — new file, Git doesn't know about it yet
- **Tracked/Unmodified** — committed, no changes
- **Modified** — changed but not staged
- **Staged** — added to index, ready to commit
- **Committed** — saved to local repo history

### The .git Directory

Everything Git needs lives in `.git/`:
- `HEAD` — pointer to the current branch/commit
- `objects/` — all commits, trees, blobs (content)
- `refs/` — branches and tags
- `config` — local repository settings
- `index` — the staging area

### Key Objects in Git

| Object | Description |
|---|---|
| **Blob** | File content (no filename, just data) |
| **Tree** | Directory listing (maps filenames to blobs/trees) |
| **Commit** | Snapshot — points to a tree + parent commit(s) + metadata |
| **Tag** | Named pointer to a commit (annotated or lightweight) |

Everything is identified by a **SHA-1 hash** (40-character hex string).

---

## 3. Setting Up Git & GitHub

### Initial Configuration

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "code --wait"   # VS Code as default editor
git config --global init.defaultBranch main       # Set default branch name
git config --list                                  # See all settings
```

Config levels (each overrides the previous):
1. `--system` — all users on the machine (`/etc/gitconfig`)
2. `--global` — current user (`~/.gitconfig`)
3. `--local` — current repo (`.git/config`)

### SSH Key Setup (GitHub Authentication)

```bash
ssh-keygen -t ed25519 -C "you@example.com"    # Generate key pair
eval "$(ssh-agent -s)"                          # Start ssh-agent
ssh-add ~/.ssh/id_ed25519                       # Add private key
cat ~/.ssh/id_ed25519.pub                       # Copy public key → paste in GitHub Settings → SSH Keys
ssh -T git@github.com                           # Test connection
```

### Initialize a Repository

```bash
git init                          # New local repo
git init my-project               # New repo in a new folder
git clone <url>                   # Clone from remote
git clone <url> my-folder         # Clone into a specific folder
git clone --depth 1 <url>         # Shallow clone (only latest snapshot)
```

---

## 4. Basic Git Commands

### Daily Workflow Commands

```bash
git status                        # Show working tree status
git add file.txt                  # Stage a specific file
git add .                         # Stage all changes in current directory
git add -p                        # Interactively stage chunks (patch mode)
git commit -m "message"           # Commit with inline message
git commit -am "message"          # Stage tracked files AND commit (skips untracked)
git commit --amend                # Modify the last commit (message or content)
git diff                          # Unstaged changes
git diff --staged                 # Staged changes (vs last commit)
git diff HEAD                     # All changes since last commit
```

### Viewing History

```bash
git log                           # Full commit history
git log --oneline                 # Condensed one-line per commit
git log --oneline --graph --all   # Visual branch graph
git log -n 5                      # Last 5 commits
git log --author="Name"           # Filter by author
git log -- file.txt               # Commits that touched a specific file
git show <hash>                   # Show a specific commit's changes
```

### File Operations

```bash
git rm file.txt                   # Remove file from working dir AND staging
git rm --cached file.txt          # Remove from staging only (keep local file)
git mv old.txt new.txt            # Rename / move a file
```

### .gitignore

```
# Example .gitignore
node_modules/
*.log
.env
.DS_Store
dist/
*.pyc
__pycache__/
```

- Create `.gitignore` at the root of the repo
- Patterns apply recursively
- `!important.log` — negate a pattern (don't ignore this)
- Global ignore: `git config --global core.excludesfile ~/.gitignore_global`

---

## 5. Branching & Merging

### Why Branches?

Branches are just lightweight movable pointers to a commit. Creating a branch is instant and costs almost nothing — it's one of Git's superpowers.

### Branch Commands

```bash
git branch                        # List local branches (* = current)
git branch -a                     # List all branches (including remote)
git branch feature/login          # Create a new branch
git switch feature/login          # Switch to branch (modern command)
git checkout feature/login        # Switch to branch (older command)
git switch -c feature/login       # Create AND switch
git checkout -b feature/login     # Create AND switch (older syntax)
git branch -d feature/login       # Delete branch (safe — won't delete unmerged)
git branch -D feature/login       # Force delete
git branch -m old-name new-name   # Rename a branch
```

### HEAD

`HEAD` is a special pointer that tells Git which commit you're currently on. Normally it points to a branch name (which points to a commit). A **detached HEAD** means HEAD points directly to a commit hash rather than a branch.

```bash
git checkout <commit-hash>        # Enter detached HEAD state
git switch -                      # Go back to previous branch
```

### Merging

```bash
git merge feature/login           # Merge feature into current branch
git merge --no-ff feature/login   # Force a merge commit (no fast-forward)
git merge --squash feature/login  # Squash all commits into one staged change
git merge --abort                 # Cancel a merge in progress
```

**Fast-Forward Merge** — happens when there's no divergence; the target branch pointer just moves forward. No merge commit created.

**3-Way Merge** — happens when branches have diverged; Git finds the common ancestor and creates a new merge commit.

### Merge Conflicts

Conflict markers in a file:
```
<<<<<<< HEAD
Your current branch's version
=======
Incoming branch's version
>>>>>>> feature/login
```

To resolve:
1. Open the file, edit to the desired content (remove all markers)
2. `git add <file>` — mark as resolved
3. `git commit` — complete the merge

---

## 6. Remote Repositories

```bash
git remote -v                          # List remotes (verbose)
git remote add origin <url>            # Add a remote named "origin"
git remote remove origin               # Remove a remote
git remote rename origin upstream      # Rename remote
git remote set-url origin <new-url>    # Change remote URL

git fetch origin                       # Download changes, don't merge
git fetch --all                        # Fetch from all remotes
git pull                               # fetch + merge (or fetch + rebase with --rebase)
git pull origin main                   # Pull specific branch
git pull --rebase                      # Fetch + rebase instead of merge

git push origin main                   # Push to remote
git push -u origin main                # Push and set upstream tracking
git push --force                       # Force push (dangerous — rewrites history)
git push --force-with-lease            # Safer force push (fails if remote has new commits)
git push origin --delete feature/old   # Delete remote branch
git push --tags                        # Push all tags
```

### Tracking Branches

When you `git clone`, `main` is automatically set to track `origin/main`. You can check:
```bash
git branch -vv                    # Shows tracking info for each branch
```

---

## 7. Pull Requests (PRs)

A Pull Request is a GitHub feature (not a Git feature) to propose merging one branch into another, with code review.

### PR Lifecycle

1. Push feature branch to GitHub
2. Open PR on GitHub (base = target branch, compare = your branch)
3. Add description, assign reviewers, link issues
4. Reviewers leave comments, request changes, or approve
5. Resolve conflicts (if any)
6. Merge PR (merge commit / squash / rebase)
7. Delete the branch (optional but good practice)

### PR Best Practices

- Keep PRs small and focused
- Write a clear description (what, why, how to test)
- Link related issues using keywords: `Closes #42`, `Fixes #42`, `Resolves #42`
- Use draft PRs for work in progress
- Request specific reviewers

### Code Review Tips (Interview)

- Review logic, not style (let linters handle style)
- Be constructive, not critical of the person
- Ask questions instead of demanding changes: "What do you think about X?"
- Approve once satisfied, request changes if blocking issues exist

---

## 8. Merge Strategies

| Strategy | What it does | When to use |
|---|---|---|
| **Merge commit** | Creates a new merge commit, preserves full history | When you want a complete audit trail |
| **Squash and merge** | Combines all PR commits into one | Clean main branch history, don't need individual commits |
| **Rebase and merge** | Replays commits on top of base, linear history | Clean linear history, still see individual commits |

```bash
# Squash example manually
git merge --squash feature/login
git commit -m "feat: add login feature"

# Rebase example manually
git checkout feature/login
git rebase main
git checkout main
git merge feature/login   # Now a fast-forward
```

---

## 9. Undoing Changes

### Before Committing

```bash
git restore file.txt               # Discard unstaged changes (modern)
git checkout -- file.txt           # Discard unstaged changes (older)
git restore --staged file.txt      # Unstage a file
git reset HEAD file.txt            # Unstage (older syntax)
git clean -fd                      # Remove untracked files and directories
```

### After Committing

```bash
git revert <hash>                  # Create a new commit that undoes a commit (SAFE)
git reset --soft HEAD~1            # Undo last commit, keep changes staged
git reset --mixed HEAD~1           # Undo last commit, keep changes unstaged (default)
git reset --hard HEAD~1            # Undo last commit, DISCARD all changes (DANGEROUS)
git reset --hard origin/main       # Reset to match remote (loses local commits)
```

### The Golden Rule

**Never rewrite history on shared/public branches.** `reset --hard` and `rebase` rewrite history. Use `revert` on shared branches — it's safe because it adds a new commit.

---

## 10. Stashing

Stash saves uncommitted changes temporarily so you can switch contexts.

```bash
git stash                          # Stash current changes
git stash push -m "WIP: login"     # Stash with a message
git stash list                     # See all stashes
git stash pop                      # Apply most recent stash + drop it
git stash apply stash@{2}          # Apply a specific stash (keep it in list)
git stash drop stash@{0}           # Delete a specific stash
git stash clear                    # Delete all stashes
git stash branch feature/wip       # Create a branch from a stash
git stash push --include-untracked # Also stash untracked files
```

---

## 11. Tags & Releases

Tags mark specific points in history (typically release versions).

```bash
git tag                            # List all tags
git tag v1.0.0                     # Lightweight tag (just a pointer)
git tag -a v1.0.0 -m "Release"    # Annotated tag (includes metadata)
git tag -a v1.0.0 <hash>          # Tag a specific commit
git push origin v1.0.0             # Push a tag
git push origin --tags             # Push all tags
git tag -d v1.0.0                  # Delete local tag
git push origin --delete v1.0.0   # Delete remote tag
```

**Annotated vs Lightweight:**
- **Lightweight** — just a name pointing to a commit
- **Annotated** — stored as a full Git object with tagger, date, and message; can be signed with GPG

**Semantic Versioning (SemVer):** `MAJOR.MINOR.PATCH`
- MAJOR — breaking changes
- MINOR — new features (backward-compatible)
- PATCH — bug fixes

---

## 12. Git Log & History

```bash
git log --oneline --graph --all --decorate   # Beautiful visual history
git log --since="2 weeks ago"
git log --until="2023-01-01"
git log --grep="fix"                          # Search commit messages
git log -S "function login"                   # Search for code changes (pickaxe)
git log --follow file.txt                     # History including renames
git blame file.txt                            # Who changed each line, when
git bisect start                              # Start binary search for a bug
git bisect bad                                # Current commit is bad
git bisect good v1.0                          # v1.0 was good
# Git checks out midpoint → you test → mark good/bad until found
git bisect reset                              # End bisect
```

### `git reflog` — Your Safety Net

Reflog records every time HEAD moves (even after resets). It's your undo button for "I accidentally deleted something."

```bash
git reflog                         # See history of HEAD movements
git checkout HEAD@{3}              # Go to where HEAD was 3 moves ago
git reset --hard HEAD@{5}         # Recover to a previous state
```

---

## 13. Rebasing

Rebase replays commits from one branch on top of another, creating a **linear history**.

```bash
git checkout feature/login
git rebase main                   # Replay feature commits on top of latest main

git rebase -i HEAD~3              # Interactive rebase: rewrite last 3 commits
```

### Interactive Rebase Options (`-i`)

| Command | Action |
|---|---|
| `pick` | Keep commit as-is |
| `reword` | Keep commit, edit message |
| `edit` | Pause at this commit to amend |
| `squash` | Combine with previous commit |
| `fixup` | Like squash, but discard this commit's message |
| `drop` | Delete this commit entirely |

### Rebase vs Merge

| | Rebase | Merge |
|---|---|---|
| History | Linear, clean | Preserves full branching history |
| Commit hashes | Changed (new commits created) | Preserved |
| Safety | Risky on shared branches | Safe on shared branches |
| Use case | Local cleanup, feature branches | Merging into shared branches |

**Never rebase commits that have been pushed to a shared remote branch.**

---

## 14. Cherry-Picking

Apply a specific commit from one branch to another.

```bash
git cherry-pick <hash>             # Apply one commit
git cherry-pick A B C              # Apply multiple commits
git cherry-pick A..C               # Apply a range (A exclusive)
git cherry-pick A^..C              # Apply a range (A inclusive)
git cherry-pick --no-commit <hash> # Apply changes without committing
git cherry-pick --abort            # Cancel
```

**Use case:** A bug fix was committed to `develop` and you need it in `main` without merging all of `develop`.

---

## 15. GitHub Flow vs Git Flow

### GitHub Flow (Simple, for Continuous Delivery)

```
main
  └── feature/login        ← branch from main
        └── (work)
        └── PR → code review → merge to main → deploy
```

Rules:
1. `main` is always deployable
2. Branch from `main` for every feature/fix
3. Open a PR early
4. Merge only after review + CI passes
5. Deploy immediately after merge

### Git Flow (Complex, for Versioned Releases)

```
main          ← production releases only, tagged
develop       ← integration branch
  ├── feature/xxx    ← branches from develop
  ├── release/1.0    ← branches from develop, merges to main + develop
  └── hotfix/1.0.1   ← branches from main, merges to main + develop
```

Branches:
- `main` — production
- `develop` — next release integration
- `feature/*` — new features (off develop)
- `release/*` — release prep (off develop)
- `hotfix/*` — urgent production fixes (off main)

**Which to use?** GitHub Flow for web apps with continuous deployment. Git Flow for products with scheduled releases (mobile apps, libraries).

### Trunk-Based Development

Everyone commits directly to `main` (trunk), using feature flags to hide unfinished work. Very short-lived branches (< 1 day). Preferred by large-scale teams (Google, Meta).

---

## 16. Forking & Open Source Contribution

### Fork Workflow

1. **Fork** the repo on GitHub (creates your own copy)
2. `git clone` your fork locally
3. Add the original as `upstream`: `git remote add upstream <original-url>`
4. Branch: `git checkout -b fix/typo`
5. Make changes, commit, push to your fork
6. Open a PR from your fork's branch → original repo's main
7. Keep your fork updated:
   ```bash
   git fetch upstream
   git merge upstream/main
   git push origin main
   ```

### Key Difference: Fork vs Clone vs Branch

| | Fork | Clone | Branch |
|---|---|---|---|
| Where | GitHub (server-side copy) | Local copy | Local/remote pointer |
| Who uses it | External contributors | Anyone | Team members with write access |
| PR goes to | Original repo | N/A | Same repo |

---

## 17. GitHub Actions (CI/CD)

GitHub Actions automates workflows triggered by events (push, PR, schedule, etc.).

### Key Concepts

- **Workflow** — YAML file in `.github/workflows/`
- **Event** — what triggers the workflow (`push`, `pull_request`, `schedule`)
- **Job** — a set of steps that run on the same runner
- **Step** — individual task (shell command or Action)
- **Action** — reusable unit (from GitHub Marketplace or custom)
- **Runner** — the machine that runs jobs (GitHub-hosted or self-hosted)

### Example Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

### Common Triggers

```yaml
on:
  push:
    branches: [main, develop]
    paths: ['src/**']           # Only trigger if src/ changes
  pull_request:
    types: [opened, synchronize]
  schedule:
    - cron: '0 9 * * 1'        # Every Monday at 9am UTC
  workflow_dispatch:            # Manual trigger
  release:
    types: [published]
```

### Secrets & Environment Variables

```yaml
env:
  NODE_ENV: production

steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}
    run: ./deploy.sh
```

Add secrets in: GitHub repo → Settings → Secrets and variables → Actions

### Job Dependencies

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: test                 # Only runs if 'test' job passes
    runs-on: ubuntu-latest
    steps: [...]
```

---

## 18. GitHub Issues & Projects

### Issues

Used to track bugs, feature requests, tasks.

- **Labels** — categorize issues (bug, enhancement, good first issue)
- **Milestones** — group issues for a release or sprint
- **Assignees** — who's responsible
- **Templates** — standardize bug reports and feature requests (`.github/ISSUE_TEMPLATE/`)

### Issue Keywords in Commits/PRs

These automatically close issues when merged to the default branch:
```
Closes #42
Fixes #42
Resolves #42
```

### GitHub Projects

Kanban-style board for project management. Can link issues and PRs across repos. Supports custom fields, views, and automation.

### CODEOWNERS

Auto-assign reviewers based on file paths:
```
# .github/CODEOWNERS
*.js        @frontend-team
src/api/    @backend-team
*.yml       @devops
```

---

## 19. GitHub Security Features

### Branch Protection Rules

Repo Settings → Branches → Add rule:
- **Require PR reviews** before merging
- **Require status checks** (CI must pass)
- **Require linear history** (no merge commits)
- **Restrict who can push** to the branch
- **Require signed commits**

### Dependabot

- **Dependabot alerts** — notifies you of vulnerable dependencies
- **Dependabot security updates** — auto-creates PRs to update vulnerable deps
- **Dependabot version updates** — auto-creates PRs to keep deps up to date

Configure in `.github/dependabot.yml`:
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Secret Scanning

GitHub automatically scans for exposed secrets (API keys, tokens) and alerts you. Partner programs notify service providers (AWS, GitHub, etc.) to revoke compromised credentials.

### Code Scanning (CodeQL)

Static analysis to find vulnerabilities in your code. Free for public repos.

### GPG Commit Signing

```bash
gpg --gen-key
gpg --list-secret-keys --keyid-format=long
git config --global user.signingkey <KEY-ID>
git config --global commit.gpgsign true
git commit -S -m "Signed commit"
```

---

## 20. Advanced Git Internals

### How a Commit is Stored

```
Commit object
  ├── tree: <hash>          → points to root Tree object
  ├── parent: <hash>        → previous commit
  ├── author: Name <email> timestamp
  └── message: "..."

Tree object
  ├── blob <hash> README.md
  ├── blob <hash> index.js
  └── tree <hash> src/

Blob object
  └── raw file content
```

All objects are stored in `.git/objects/` as compressed files, named by their SHA-1 hash.

### Packfiles

Git periodically runs `git gc` (garbage collection) to compress loose objects into packfiles (`.git/objects/pack/`) using delta compression.

### `git worktree`

Work on multiple branches simultaneously without switching:
```bash
git worktree add ../hotfix hotfix/1.0.1   # Check out branch in new directory
git worktree list
git worktree remove ../hotfix
```

### `git submodule`

Include another Git repo inside your repo:
```bash
git submodule add <url> path/to/submodule
git submodule update --init --recursive    # Clone with submodules
git submodule foreach git pull             # Update all submodules
```

### `git subtree`

Alternative to submodules; merges another repo's history into a subdirectory:
```bash
git subtree add --prefix=lib/ <url> main --squash
git subtree pull --prefix=lib/ <url> main --squash
```

### Hooks

Scripts that run automatically on Git events. Located in `.git/hooks/`:

| Hook | Trigger |
|---|---|
| `pre-commit` | Before commit (run linters) |
| `commit-msg` | Validate commit message |
| `pre-push` | Before push (run tests) |
| `post-merge` | After merge (install deps) |
| `pre-rebase` | Before rebase |

Use **Husky** (npm) or **pre-commit** (Python) to manage hooks across teams.

### Git Attributes

`.gitattributes` — control how Git handles files:
```
*.png binary                   # Treat as binary (no diff)
*.sh eol=lf                    # Force LF line endings
*.md diff=markdown             # Use markdown differ
```

---

## 21. Common Interview Questions & Answers

### Q: What is the difference between `git fetch` and `git pull`?

**A:** `git fetch` downloads changes from the remote to your local tracking branches (e.g., `origin/main`) but does NOT merge them into your working branch. You can review changes before integrating. `git pull` is a shortcut for `git fetch` + `git merge` (or `git fetch` + `git rebase` if configured). Use `fetch` when you want to inspect changes first; use `pull` when you trust the remote and want to update immediately.

---

### Q: What is the difference between `git reset` and `git revert`?

**A:** Both undo changes, but differently:
- `git reset` moves the branch pointer backward, effectively removing commits from history. It rewrites history and is **unsafe on shared branches**.
- `git revert` creates a new commit that undoes the changes of a previous commit. History is preserved and it's **safe on shared branches**.

Use `revert` on `main`/shared branches. Use `reset` only on local/private branches.

---

### Q: What is a detached HEAD state?

**A:** Normally, HEAD points to a branch name (e.g., `main`), which in turn points to a commit. In detached HEAD state, HEAD points directly to a commit hash instead of a branch. This happens when you `git checkout <commit-hash>`. New commits made in this state aren't on any branch and can be lost. To save work, create a branch: `git switch -c new-branch`.

---

### Q: What is rebasing and when should you use it?

**A:** Rebasing replays your commits on top of another branch's tip, creating a linear history. It rewrites commit history (new SHA hashes). Use it to:
- Clean up messy local commits before opening a PR
- Update a feature branch with the latest main without a merge commit
- Squash/reorder commits with interactive rebase

**Never rebase shared/public branches** — it rewrites history and causes conflicts for teammates.

---

### Q: How do you resolve a merge conflict?

**A:**
1. Git marks conflicting files and pauses the merge
2. Open each conflicting file — look for `<<<<<<<`, `=======`, `>>>>>>>` markers
3. Manually edit the file to the desired state (remove all markers)
4. `git add <file>` to mark as resolved
5. `git commit` to complete the merge (or `git merge --continue`)
6. If you want to abort: `git merge --abort`

---

### Q: What is the difference between a fork and a branch?

**A:** A **branch** is a pointer within the same repository — all team members with access can see it. A **fork** is a server-side copy of the entire repository under a different user's account. Forks are used by external contributors who don't have write access to the original repo. They make changes in their fork and open a PR to the original. Branches are used by team members who have write access.

---

### Q: What does `git stash` do?

**A:** `git stash` saves your uncommitted changes (both staged and unstaged) onto a stack and reverts the working directory to the last commit. This lets you quickly switch branches without committing half-done work. `git stash pop` restores the most recent stash and removes it from the stack. `git stash apply` restores it but keeps it in the stash list.

---

### Q: What is `cherry-pick`?

**A:** `git cherry-pick <hash>` applies the changes from a specific commit to your current branch, creating a new commit. It's useful when you want a specific fix or feature commit from another branch without merging the entire branch. Common use: a bug fix committed to `develop` that urgently needs to go to `main`/production.

---

### Q: What are GitHub Actions?

**A:** GitHub Actions is GitHub's built-in CI/CD platform. You define workflows in YAML files in `.github/workflows/`. Workflows are triggered by events (push, PR, schedule, etc.) and consist of jobs, which contain steps (shell commands or pre-built Actions). Common uses: running tests on every PR, building and deploying on merge to main, scheduled tasks, security scanning.

---

### Q: How do you undo the last commit without losing your changes?

**A:** `git reset --soft HEAD~1` — moves HEAD back one commit but keeps all changes staged. `git reset --mixed HEAD~1` — moves HEAD back and unstages changes (keeps files). `git reset --hard HEAD~1` — moves HEAD back AND discards all changes (dangerous). If the commit was already pushed, use `git revert HEAD` instead to create a new undo commit safely.

---

### Q: What is `git bisect`?

**A:** `git bisect` uses binary search to find which commit introduced a bug. You tell Git a known "bad" commit (current) and a known "good" commit (where the bug didn't exist). Git checks out the midpoint, you test and tell it `good` or `bad`, and it narrows down until it finds the exact commit that introduced the bug. Can be automated with `git bisect run <test-script>`.

---

### Q: What is the difference between `git merge --ff`, `--no-ff`, and `--squash`?

**A:**
- `--ff` (fast-forward, default when possible): If the target branch hasn't diverged, just move the pointer forward. No merge commit.
- `--no-ff`: Always creates a merge commit, even if a fast-forward is possible. Preserves branch history visually.
- `--squash`: Combines all commits from the source branch into one staged change, then you commit manually. Clean history but loses individual commit messages from the branch.

---

## Quick Reference Cheat Sheet

```bash
# SETUP
git config --global user.name/email "..."
git init / git clone <url>

# DAILY
git status / git add . / git commit -m "msg"
git pull / git push

# BRANCHES
git switch -c feature/x    # create + switch
git merge feature/x        # merge into current
git branch -d feature/x    # delete after merge

# UNDO
git restore file.txt        # discard unstaged change
git reset --soft HEAD~1    # undo commit, keep staged
git revert <hash>          # safe undo (new commit)

# INSPECT
git log --oneline --graph --all
git diff / git diff --staged
git blame file.txt

# REMOTE
git remote add origin <url>
git push -u origin main
git fetch / git pull --rebase

# ADVANCED
git rebase -i HEAD~3       # interactive rebase
git cherry-pick <hash>     # apply specific commit
git stash / git stash pop  # temporary save
git bisect start           # find bug commit
git reflog                 # recover lost commits
```

---

*Last updated: 2025 | Covers Git 2.x & GitHub current features*
