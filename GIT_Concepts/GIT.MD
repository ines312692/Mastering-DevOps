# Git Interview Questions and Answers

## Table of Contents

1. [Basic Concepts](#basic-concepts)
2. [Branching and Merging](#branching-and-merging)
3. [Undoing Changes](#undoing-changes)
4. [Advanced Git Operations](#advanced-git-operations)
5. [Git Workflows](#git-workflows)
6. [Best Practices](#best-practices)

---

## Basic Concepts

### Q1: What is Git and how does it differ from other version control systems?

**Answer:**

Git is a distributed version control system (DVCS) where every developer has a full copy of the repository history locally.

| Centralized VCS (SVN) | Distributed VCS (Git) |
|-----------------------|-----------------------|
| Single central server | Every clone is full repository |
| Requires network for most operations | Most operations are local |
| Single point of failure | No single point of failure |
| Linear history | Branching is lightweight |

Key Git characteristics:
- Snapshots, not differences (stores complete state)
- Nearly every operation is local
- Integrity through SHA-1 checksums
- Generally only adds data

---

### Q2: Explain the three states in Git.

**Answer:**

Git has three main states for files:

```
Working Directory    Staging Area (Index)    Repository (.git)
      |                     |                      |
      |--- git add -------->|                      |
      |                     |--- git commit ------>|
      |<----------- git checkout ------------------|
```

1. **Modified**: Changed file in working directory, not staged
2. **Staged**: Marked modified file to go into next commit
3. **Committed**: Data safely stored in local database

```bash
# Check file states
git status

# Stage files
git add <file>
git add .

# Commit staged files
git commit -m "message"
```

---

### Q3: What is the difference between `git fetch` and `git pull`?

**Answer:**

| git fetch | git pull |
|-----------|----------|
| Downloads changes from remote | Downloads AND merges changes |
| Does not modify working directory | Modifies working directory |
| Safe operation | May cause merge conflicts |
| Updates remote-tracking branches | Updates current branch |

```bash
# Fetch only (safe)
git fetch origin
git diff main origin/main  # Review changes
git merge origin/main      # Then merge manually

# Pull (fetch + merge in one step)
git pull origin main

# Pull with rebase instead of merge
git pull --rebase origin main
```

**Best practice**: Use `git fetch` followed by review, then merge for critical branches.

---

## Branching and Merging

### Q4: Explain the difference between `git merge` and `git rebase`.

**Answer:**

| Merge | Rebase |
|-------|--------|
| Creates merge commit | Rewrites commit history |
| Preserves complete history | Creates linear history |
| Non-destructive | Destructive (changes SHAs) |
| Shows when branches joined | Cleaner log |

```bash
# Merge - creates merge commit
git checkout main
git merge feature

# Rebase - replays commits on top of main
git checkout feature
git rebase main

# Interactive rebase - squash, edit, reorder commits
git rebase -i HEAD~3
```

**Golden rule**: Never rebase public/shared branches.

Visual comparison:

```
MERGE:
main:    A---B---C-------M
              \         /
feature:       D---E---F

REBASE:
main:    A---B---C
                  \
feature:           D'--E'--F'
```

---

### Q5: What is a fast-forward merge?

**Answer:**

A fast-forward merge occurs when the target branch has not diverged from the source branch. Git simply moves the pointer forward.

```
Before:
main:    A---B---C
                  \
feature:           D---E

After fast-forward merge:
main:    A---B---C---D---E (pointer moved forward)
```

```bash
# Fast-forward merge (default when possible)
git checkout main
git merge feature

# Force merge commit even when fast-forward is possible
git merge --no-ff feature

# Only merge if fast-forward is possible
git merge --ff-only feature
```

---

### Q6: How do you resolve merge conflicts?

**Answer:**

```bash
# 1. Start merge
git merge feature-branch
# CONFLICT message appears

# 2. See conflicting files
git status

# 3. Open file and resolve manually
<<<<<<< HEAD
current branch code
=======
incoming branch code
>>>>>>> feature-branch

# 4. After resolving, stage file
git add resolved-file.txt

# 5. Complete merge
git commit -m "Merge feature-branch, resolved conflicts"

# Alternative: use merge tool
git mergetool

# Abort merge if needed
git merge --abort
```

**Prevention strategies:**
- Small, frequent merges
- Communication between team members
- Feature flags for parallel work
- Clear code ownership

---

## Undoing Changes

### Q7: How do you undo commits in Git?

**Answer:**

```bash
# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, keep changes unstaged (default)
git reset --mixed HEAD~1

# Undo last commit, discard changes completely
git reset --hard HEAD~1

# Create new commit that undoes changes (safe for shared branches)
git revert <commit-sha>

# Undo a pushed commit safely
git revert HEAD
git push

# Amend last commit message
git commit --amend -m "New message"

# Amend last commit with new files
git add forgotten-file.txt
git commit --amend --no-edit
```

| Command | Staged | Working Dir | Safe for shared |
|---------|--------|-------------|-----------------|
| reset --soft | Keeps | Keeps | No |
| reset --mixed | Clears | Keeps | No |
| reset --hard | Clears | Clears | No |
| revert | N/A | N/A | Yes |

---

### Q8: What is `git stash` and when do you use it?

**Answer:**

Stash temporarily saves uncommitted changes without committing.

```bash
# Save current changes
git stash
git stash save "WIP: feature description"

# List stashes
git stash list

# Apply most recent stash (keep in stash)
git stash apply

# Apply and remove from stash
git stash pop

# Apply specific stash
git stash apply stash@{2}

# Create branch from stash
git stash branch new-branch stash@{0}

# Drop a stash
git stash drop stash@{0}

# Clear all stashes
git stash clear

# Stash including untracked files
git stash -u

# Stash including ignored files
git stash -a
```

**Use cases:**
- Switch branches with uncommitted work
- Pull changes when you have local modifications
- Temporarily set aside experimental changes
- Clean working directory for testing

---

## Advanced Git Operations

### Q9: Explain `git cherry-pick`.

**Answer:**

Cherry-pick applies specific commit(s) to current branch without merging entire branch.

```bash
# Apply single commit
git cherry-pick <commit-sha>

# Apply multiple commits
git cherry-pick <sha1> <sha2> <sha3>

# Apply range of commits
git cherry-pick <start-sha>^..<end-sha>

# Cherry-pick without committing
git cherry-pick -n <commit-sha>

# Continue after resolving conflicts
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort
```

**Use cases:**
- Backport bug fix to release branch
- Apply specific feature commit without merging entire branch
- Recover commit from deleted branch
- Apply hotfix to multiple branches

---

### Q10: What are Git hooks? Provide examples.

**Answer:**

Git hooks are scripts that run automatically at specific Git events.

**Location:** `.git/hooks/` (local) or shared via tools

| Hook | Trigger | Use Case |
|------|---------|----------|
| pre-commit | Before commit | Linting, formatting |
| commit-msg | After message entered | Validate message format |
| pre-push | Before push | Run tests |
| post-merge | After merge | Install dependencies |
| pre-rebase | Before rebase | Prevent rebasing |

**Example pre-commit hook:**

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run linting
npm run lint
if [ $? -ne 0 ]; then
  echo "Linting failed. Commit aborted."
  exit 1
fi

# Run tests
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed. Commit aborted."
  exit 1
fi

exit 0
```

**Sharing hooks with team using Husky (npm):**

```bash
npm install husky --save-dev
npx husky install
npx husky add .husky/pre-commit "npm run lint"
```

**Using pre-commit framework (Python):**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: check-yaml
      - id: check-json
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.16.1
    hooks:
      - id: gitleaks
```

---

### Q11: Explain Git submodules vs Git subtree.

**Answer:**

| Submodules | Subtree |
|------------|---------|
| Link to external repo at specific commit | Copies external repo into subdirectory |
| Separate .git history | Merged into main repo history |
| Requires explicit update commands | Simpler workflow |
| `.gitmodules` file tracks repos | No extra metadata files |
| Clone needs `--recursive` | Regular clone works |

**Submodule commands:**

```bash
# Add submodule
git submodule add https://github.com/org/lib.git libs/lib

# Clone with submodules
git clone --recursive <repo-url>

# Update submodules
git submodule update --init --recursive

# Update to latest commit
git submodule update --remote
```

**Subtree commands:**

```bash
# Add subtree
git subtree add --prefix=libs/lib https://github.com/org/lib.git main --squash

# Pull updates
git subtree pull --prefix=libs/lib https://github.com/org/lib.git main --squash

# Push changes back
git subtree push --prefix=libs/lib https://github.com/org/lib.git main
```

---

### Q12: What is `git reflog` and when is it useful?

**Answer:**

Reflog records all reference updates in the local repository, including commits that are no longer referenced.

```bash
# View reflog
git reflog

# View reflog for specific branch
git reflog show feature-branch

# Recover deleted branch
git reflog
# Find SHA before deletion
git checkout -b recovered-branch <sha>

# Recover after hard reset
git reflog
git reset --hard HEAD@{2}

# Clean old reflog entries
git reflog expire --expire=30.days --all
```

**Output example:**

```
abc1234 HEAD@{0}: commit: Add new feature
def5678 HEAD@{1}: checkout: moving from main to feature
ghi9012 HEAD@{2}: reset: moving to HEAD~1
jkl3456 HEAD@{3}: commit: Previous commit
```

**Use cases:**
- Recover deleted commits
- Undo accidental reset --hard
- Find lost work after rebase
- Debug branch history

---

## Git Workflows

### Q13: Explain GitFlow branching strategy.

**Answer:**

GitFlow uses multiple long-lived branches for different purposes.

```
        hotfix
          |
main -----+-----+-----+-----+-----+-----
          |     |     |     |
release --+     +--+--+     |
                   |        |
develop ----+------+--------+------+----
            |                      |
feature ----+----------------------+
```

**Branch types:**

| Branch | Purpose | Created from | Merged to |
|--------|---------|--------------|-----------|
| main | Production code | - | - |
| develop | Integration branch | main | main, release |
| feature/* | New features | develop | develop |
| release/* | Release preparation | develop | main, develop |
| hotfix/* | Production fixes | main | main, develop |

**Commands:**

```bash
# Start feature
git checkout develop
git checkout -b feature/new-feature

# Finish feature
git checkout develop
git merge --no-ff feature/new-feature
git branch -d feature/new-feature

# Start release
git checkout develop
git checkout -b release/1.0.0

# Finish release
git checkout main
git merge --no-ff release/1.0.0
git tag -a v1.0.0 -m "Version 1.0.0"
git checkout develop
git merge --no-ff release/1.0.0

# Hotfix
git checkout main
git checkout -b hotfix/critical-fix
# After fix
git checkout main
git merge --no-ff hotfix/critical-fix
git checkout develop
git merge --no-ff hotfix/critical-fix
```

---

### Q14: Explain Trunk-Based Development.

**Answer:**

All developers work on a single branch (main/trunk) with short-lived feature branches.

**Principles:**
- Small, frequent commits to main
- Feature branches live less than 1-2 days
- Feature flags for incomplete work
- CI runs on every commit
- Main is always deployable

**Comparison:**

```
GitFlow:
  main ──────────────────────────────────
         \                    /
  develop ─────────────────────
           \        /
  feature ────────

Trunk-Based:
  main ───*───*───*───*───*───*───*───*──
             \   /
  feature     ─*─  (short-lived, <2 days)
```

**Advantages:**
- Faster integration
- Less merge conflicts
- Continuous deployment friendly
- Simpler mental model
- Reduced integration risk

**Requirements:**
- Strong CI/CD pipeline
- Feature flags infrastructure
- Code review process
- Automated testing

---

## Best Practices

### Q15: How do you handle secrets in Git?

**Answer:**

**Never commit secrets.** Here are solutions:

1. **Environment variables** - inject at runtime

```bash
# .env (add to .gitignore)
DATABASE_PASSWORD=secret123

# In application
db_password = os.environ.get('DATABASE_PASSWORD')
```

2. **Secret management tools**
    - HashiCorp Vault
    - AWS Secrets Manager
    - Azure Key Vault
    - Google Secret Manager

3. **Encrypted secrets**
    - SOPS (Secrets OPerationS)
    - git-crypt
    - Sealed Secrets (Kubernetes)

4. **Pre-commit hooks for detection**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.16.1
    hooks:
      - id: gitleaks
```

5. **.gitignore patterns**

```gitignore
# Secrets and credentials
.env
.env.*
*.pem
*.key
credentials.json
secrets.yaml
```

**If secrets are accidentally committed:**

```bash
# Remove from history (use BFG or git-filter-repo)
bfg --delete-files secrets.json
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Rotate the compromised credentials immediately
```

---

### Q16: What is a good commit message format?

**Answer:**

**Conventional Commits format:**

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- feat: New feature
- fix: Bug fix
- docs: Documentation
- style: Formatting
- refactor: Code restructuring
- test: Adding tests
- chore: Maintenance

**Examples:**

```
feat(auth): add OAuth2 login support

Implement Google and GitHub OAuth2 providers.
Users can now sign in using their social accounts.

Closes #123

---

fix(api): resolve race condition in user creation

The user service was not properly handling concurrent
requests, leading to duplicate entries.

- Add database transaction
- Implement optimistic locking
- Add retry logic

Fixes #456
```

**Rules:**
- Subject line max 50 characters
- Use imperative mood ("Add feature" not "Added feature")
- Body wrapped at 72 characters
- Explain what and why, not how

---

### Q17: How do you write a good .gitignore file?

**Answer:**

```gitignore
# Operating System
.DS_Store
Thumbs.db
*.swp
*~

# IDEs and Editors
.idea/
.vscode/
*.sublime-*
*.iml

# Dependencies
node_modules/
vendor/
venv/
__pycache__/
*.pyc

# Build outputs
dist/
build/
target/
*.jar
*.war
*.class

# Logs
*.log
logs/
npm-debug.log*

# Environment and secrets
.env
.env.*
*.pem
*.key
credentials.json

# Testing
coverage/
.coverage
.pytest_cache/
.nyc_output/

# Temporary files
tmp/
temp/
*.tmp
*.bak

# Package manager locks (choose one per project)
# package-lock.json  # Uncomment if using yarn
# yarn.lock          # Uncomment if using npm
```

**Tips:**
- Use gitignore.io for language-specific templates
- Put project-specific ignores in repo, personal ones in global gitignore
- Use `git check-ignore -v <file>` to debug

```bash
# Set global gitignore
git config --global core.excludesfile ~/.gitignore_global
```

---

## Summary

| Topic | Key Commands |
|-------|--------------|
| Staging | git add, git reset |
| Committing | git commit, git commit --amend |
| Branching | git branch, git checkout, git switch |
| Merging | git merge, git rebase |
| Remote | git fetch, git pull, git push |
| Undoing | git reset, git revert, git checkout |
| Stashing | git stash, git stash pop |
| History | git log, git reflog, git blame |
| Inspection | git diff, git status, git show |