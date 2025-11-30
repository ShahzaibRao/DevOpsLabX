# 058: Git

---

https://drive.google.com/file/d/1xfT6raH1ryvW_56W2L5FI-180_hUfyCs/view?usp=sharing

## 1. Introduction to Git

### What is Git?

- **Distributed Version Control System (DVCS)**: Tracks changes to files over time
- **Created by Linus Torvalds** in 2005 for Linux kernel development
- **Key Features**:
    - Distributed (every developer has full repository history)
    - Fast and efficient
    - Supports non-linear development (branches, merges)
    - Data integrity (cryptographic hashing)

### Git vs GitHub vs GitLab

| Tool | Purpose |
| --- | --- |
| **Git** | Local version control system |
| **GitHub** | Cloud-based Git repository hosting + collaboration |
| **GitLab** | Alternative to GitHub with built-in CI/CD |

---

## 2. Git Installation and Configuration

### Install Git

```bash
# RHEL/CentOS/Fedora
sudo dnf install git -y

# Ubuntu/Debian
sudo apt install git -y

# Verify installation
git --version

```

### Configure Git Identity

```bash
# Set global identity (required)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default editor
git config --global core.editor "vim"

# Set default branch name
git config --global init.defaultBranch main

# View all configurations
git config --list

```

### Configuration Files

- **System**: `/etc/gitconfig` (all users)
- **Global**: `~/.gitconfig` (current user)
- **Local**: `.git/config` (current repository)

---

## 3. Basic Git Workflow

### Initialize Repository

```bash
# Create new repository
mkdir my-project
cd my-project
git init

# Or clone existing repository
git clone <https://github.com/username/repository.git>

```

### Basic Commands

```bash
# Check status
git status

# Add files to staging area
git add filename.txt          # Single file
git add .                     # All files
git add *.txt                 # Pattern matching

# Commit changes
git commit -m "Descriptive commit message"

# View commit history
git log
git log --oneline            # Compact view
git log --graph              # Visual branch history

```

### The Three States of Git

1. **Working Directory**: Files on your filesystem
2. **Staging Area (Index)**: Files prepared for next commit
3. **Repository**: Committed snapshots stored in `.git` directory

---

## 4. Branching and Merging

### Branch Management

```bash
# List branches
git branch                    # Local branches
git branch -a                 # All branches (local + remote)

# Create branch
git branch feature-branch

# Switch to branch
git checkout feature-branch
# OR (newer syntax)
git switch feature-branch

# Create and switch to new branch
git checkout -b feature-branch
# OR
git switch -c feature-branch

# Delete branch
git branch -d feature-branch  # Safe delete
git branch -D feature-branch  # Force delete

```

### Merging Branches

```bash
# Switch to target branch (usually main)
git checkout main

# Merge feature branch
git merge feature-branch

# Resolve merge conflicts (if any)
# 1. Edit conflicted files
# 2. git add resolved-file.txt
# 3. git commit

```

### Merge Strategies

- **Fast-forward**: Linear history (no merge commit)
- **Recursive**: Creates merge commit (default when needed)
- **Squash**: Combine all commits into one

---

## 5. Remote Repositories

### Remote Management

```bash
# View remotes
git remote -v

# Add remote
git remote add origin <https://github.com/username/repository.git>

# Remove remote
git remote remove origin

# Rename remote
git remote rename origin upstream

```

### Push and Pull

```bash
# Push local commits to remote
git push origin main

# Push new branch
git push -u origin feature-branch  # -u sets upstream tracking

# Pull changes from remote
git pull origin main

# Fetch without merging
git fetch origin

```

### Common Remote Scenarios

```bash
# Clone and set up tracking
git clone <https://github.com/user/repo.git>
git checkout -b feature origin/feature

# Update fork from upstream
git remote add upstream <https://github.com/original/repo.git>
git fetch upstream
git checkout main
git merge upstream/main

```

---

## 6. Undoing Changes

### Different Levels of Undo

```bash
# Unstage file (keep changes in working directory)
git restore --staged filename.txt

# Discard changes in working directory
git restore filename.txt

# Revert to last commit (discard all changes)
git reset --hard HEAD

# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Create reverse commit (safe for shared history)
git revert HEAD

```

### Time Travel with Reset

| Command | Effect |
| --- | --- |
| `git reset --soft HEAD~1` | Undo commit, keep changes staged |
| `git reset --mixed HEAD~1` | Undo commit, unstage changes (default) |
| `git reset --hard HEAD~1` | Undo commit and changes (destructive) |

> ⚠️ Warning: Never use --hard on shared branches!
> 

---

## 7. Advanced Git Features

### Stashing Changes

```bash
# Save current changes temporarily
git stash

# List stashes
git stash list

# Apply latest stash
git stash pop

# Apply specific stash
git stash apply stash@{1}

# Drop stash
git stash drop stash@{1}

```

### Tagging Releases

```bash
# Create lightweight tag
git tag v1.0

# Create annotated tag (recommended)
git tag -a v1.0 -m "Version 1.0 release"

# Push tags
git push origin v1.0
git push origin --tags  # Push all tags

```

### Cherry-Picking Commits

```bash
# Apply specific commit to current branch
git cherry-pick <commit-hash>

# Cherry-pick range of commits
git cherry-pick <start>..<end>

```

---

## 8. Git Best Practices

### Commit Messages

**Good commit message format**:

```
feat: add user authentication

- Implement login/logout functionality
- Add password validation
- Update documentation

```

**Rules**:

- Use imperative mood ("Add feature" not "Added feature")
- Keep first line under 50 characters
- Separate subject from body with blank line
- Reference issues: `Closes #123`

### Branch Naming

- `main` / `master`: Production code
- `develop`: Integration branch
- `feature/feature-name`: New features
- `bugfix/issue-description`: Bug fixes
- `release/v1.0.0`: Release preparation

### Workflow Strategies

| Workflow | Best For |
| --- | --- |
| **Centralized** | Small teams, simple projects |
| **Feature Branch** | Most common, good for medium teams |
| **GitFlow** | Release-heavy projects |
| **Forking** | Open source contributions |

---

## 9. Git Configuration Tips

### Useful Aliases

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'restore --staged'
git config --global alias.last 'log -1 HEAD'

```

### Ignoring Files

```bash
# Create .gitignore file
echo "*.log" >> .gitignore
echo "node_modules/" >> .gitignore
echo ".env" >> .gitignore

# Global .gitignore
git config --global core.excludesfile ~/.gitignore_global

```

### Common .gitignore Patterns

```
# OS generated
.DS_Store
Thumbs.db

# Build artifacts
*.o
*.so
*.dll

# Dependencies
node_modules/
vendor/

# Environment variables
.env
.env.local

# IDE
.vscode/
.idea/
*.swp

```

---

## 10. Troubleshooting Common Issues

### Merge Conflicts

**Steps to resolve**:

1. Identify conflicted files: `git status`
2. Edit files to resolve conflicts (look for `<<<<<<<`, `=======`, `>>>>>>>`)
3. Stage resolved files: `git add resolved-file.txt`
4. Complete merge: `git commit`

### Detached HEAD State

**Cause**: Checking out a specific commit instead of branch
**Fix**:

```bash
# Create new branch from current commit
git checkout -b new-branch-name

# Or return to main branch
git checkout main

```

### Recover Lost Commits

```bash
# View reflog (history of HEAD movements)
git reflog

# Reset to lost commit
git reset --hard <commit-hash>

```

### Large Files Issue

```bash
# Install Git LFS (Large File Storage)
git lfs install
git lfs track "*.psd"
git add .gitattributes

```

---

## Key Commands Cheat Sheet

| Category | Command |
| --- | --- |
| **Setup** | `git config --global user.name "Name"` |
| **Initialize** | `git init` / `git clone <url>` |
| **Status** | `git status` / `git log` |
| **Stage** | `git add .` / `git restore --staged file` |
| **Commit** | `git commit -m "message"` |
| **Branch** | `git branch` / `git checkout -b branch` |
| **Merge** | `git merge branch` |
| **Remote** | `git remote -v` / `git push origin main` |
| **Undo** | `git reset --hard HEAD~1` / `git revert HEAD` |
| **Stash** | `git stash` / `git stash pop` |

---

## Summary Workflow

```bash
# 1. Initialize
git init
git config --global user.name "Your Name"

# 2. Basic workflow
echo "Hello" > file.txt
git add file.txt
git commit -m "Initial commit"

# 3. Branching
git checkout -b feature
# Make changes
git add .
git commit -m "Add feature"

# 4. Merge
git checkout main
git merge feature

# 5. Remote
git remote add origin <url>
git push -u origin main

```

> 💡 Golden Rule:
> 
> 
> **Commit early, commit often**! Always:
> 
> - Write **meaningful commit messages**
> - **Branch for features** (never work directly on main)
> - **Pull before push** to avoid conflicts