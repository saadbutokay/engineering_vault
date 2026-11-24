Imagine writing a 10,000 line codebase and then:
```
Scenario 1: You change something. It breaks everything.
            You can't remember what you changed.
            You wish you could go back to yesterday's version.

Scenario 2: You and a teammate both edit the same file.
            You overwrite each other's work.
            Someone's changes are lost forever.

Scenario 3: Your laptop dies.
            All your code is gone.
```
**Git solves ALL three of these problems.**

Git is a system that:
  - Takes snapshots of your code over time
  - Lets you go back to ANY previous snapshot
  - Lets multiple people work on the same code
  - Stores your code safely on the internet (GitHub)
  - Tracks WHO changed WHAT and WHEN

[Click this](https://youtu.be/8JJ101D3knE) to watch it in YouTube. and use my [[Git Cheatsheet]], or the [Official Cheatsheet](https://git-scm.com/cheat-sheet).

---
## 1. Core Concepts
Before commands, understand the mental model.
### 1.1 The Three States of Git
```
Your files exist in 3 possible states:
┌─────────────────────────────────────────────────────┐
│                                                     │
│  WORKING DIRECTORY    STAGING AREA      REPOSITORY  │
│  (your files)         (ready room)      (history)   │
│                                                     │
│ main.py ──►git add ──►main.py ──►git commit ──►store│
│  (modified)           (staged)            (snapshot)│
│                                                     │
└─────────────────────────────────────────────────────┘

Working Directory: Files you're currently editing
Staging Area:      Files you've selected for next snapshot
Repository:        Permanent history of all snapshots
```

**Real world analogy:**
```
Taking a group photo:
Working Directory = people walking around the room
Staging Area      = people you've asked to stand for the photo
Commit            = taking the actual photo (permanent)

git add     = "Hey you, stand here for the photo"
git commit  = "Click" - photo taken forever
```

### 1.2 What is a Repository?
A repository (repo) is just a folder that Git is tracking.
```
my_project/          ← this is your repo
├── .git/            ← Git stores ALL history in here
│   ├── commits/     (never touch this folder manually)
│   ├── branches/
│   └── config
├── main.py
└── README.md

The .git/ folder = Git's brain
Delete it = lose all history
```

### 1.3 What is a Commit?
A commit is a snapshot of your code at a moment in time.
```
Each commit has:
  - A unique ID:  a3f8d91  (called a hash)
  - A message:    "Add user login endpoint"
  - Author:       "John Doe"
  - Timestamp:    "2024-01-15 14:32:00"
  - The changes:  exactly what was added/removed
```

```
Your commit history looks like:

  a3f8d91  "Add user login endpoint"        ← latest
  b7c2e44  "Create users table migration"
  f9a1d23  "Initialize FastAPI project"
  e4b8c12  "Initial commit"                 ← oldest
```

---
## 2. Installing Git

**Check if Git is installed:**
```bash
git --version
# Output: git version 2.43.0
```

**Install if not present:**

**Windows:**
1. Go to [git-scm.com/download/win](https://git-scm.com/download/win)
2. Download and install Git for Windows
3. During install, choose:
   - "Git from command line and also from 3rd-party software"
   - "Use VS Code as Git's default editor"
   - "Override the default branch name" → type: `main`
4. Verify: open new terminal → `git --version`

**Mac:**
```bash
# Using Homebrew:
brew install git

# OR install Xcode Command Line Tools:
xcode-select --install
```

**Linux (Ubuntu):**
```bash
sudo apt update
sudo apt install git
```

---
## 3. First-Time Git Setup

**Do this ONCE after installing Git:**
```bash
# Set your name (shows up in every commit you make)
git config --global user.name "Your Full Name"

# Set your email (must match your GitHub email)
git config --global user.email "you@example.com"

# Set default branch name to "main" (modern standard)
git config --global init.defaultBranch main

# Set VS Code as your default Git editor
git config --global core.editor "code --wait"

# Set line endings (important for cross-platform teams)
# Mac/Linux:
git config --global core.autocrlf input
# Windows:
git config --global core.autocrlf true

# Verify all settings
git config --global --list
```

---
## 4. Core Git Commands - The Daily Workflow

### 4.1 Starting a Repository
```bash
# Option A: Start fresh in an existing folder
cd my_project
git init

# Git creates a .git/ folder
# Output: Initialized empty Git repository in /your/path/.git/

# Option B: Download an existing repo from GitHub
git clone https://github.com/username/repository-name.git

# Clone into a specific folder name
git clone https://github.com/username/repo.git my_folder_name
```

### 4.2 Checking Status

```bash
git status
```

**What it shows:**
```
# Output example:
On branch main

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   main.py          ← you changed this

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        new_feature.py           ← new file Git doesn't know about

nothing added to commit but untracked files present
```
**Run `git status` constantly. It tells you exactly where you are.**

### 4.3 The `add` → Commit Cycle
```bash
# Stage a specific file
git add main.py

# Stage multiple files
git add main.py utils.py models.py

# Stage ALL changed files
git add .

# Check what's staged
git status

# Commit with a message
git commit -m "Add user authentication endpoint"

# Stage and commit in one command (only for tracked files)
git commit -am "Fix typo in login function"
```

### 4.4 Viewing History
```bash
# See all commits
git log

# Output:
# commit a3f8d91b2c4e6f8a0d1e3f5b7c9e0a2d4f6b8c0
# Author: Your Name <you@email.com>
# Date:   Mon Jan 15 14:32:00 2024
#
#     Add user login endpoint

# Compact one-line view (use this more)
git log --oneline

# Output:
# a3f8d91 Add user login endpoint
# b7c2e44 Create users table migration
# f9a1d23 Initialize FastAPI project

# Visual graph (great for branches)
git log --oneline --graph --all

# See what changed in a specific commit
git show a3f8d91
```

### 4.5 Seeing What Changed
```bash
# See changes in working directory (not yet staged)
git diff

# See changes that ARE staged (ready to commit)
git diff --staged

# See changes between two commits
git diff a3f8d91 b7c2e44
```

```bash
# Output example:
 --- a/main.py
 +++ b/main.py
 @@ -10,6 +10,10 @@
  def get_users():
 -    return []
 +    users = db.query(User).all()
 +    return users

 Red lines (-)  = removed
 Green lines (+) = added
```

### 4.6 Undoing Mistakes
```bash
# Undo changes to a file (before staging)
# Restore file to last commit state
git restore main.py

# Unstage a file (remove from staging area)
git restore --staged main.py

# Undo last commit but KEEP the changes
git reset --soft HEAD~1

# Undo last commit and DISCARD the changes
# ⚠️ DANGEROUS — changes are gone
git reset --hard HEAD~1

# Create a new commit that reverses a previous one
# (safe way to undo — doesn't delete history)
git revert a3f8d91
```

**Visual:**
```
git reset --soft HEAD~1:
  Before: [commit A] [commit B] [commit C] ← HEAD
  After:  [commit A] [commit B] ← HEAD
          (C's changes back in working directory)

git revert:
  Before: [commit A] [commit B] [commit C] ← HEAD
  After:  [commit A] [commit B] [commit C] [undo of C] ← HEAD
          (history preserved, safe for shared repos)
```

---
## 5. Branching
The default branch is called "main". It's your stable, working code. A branch is a separate copy where you can work without affecting main.
```
main:     A → B → C → D
                ↘
feature:            E → F → G
                          ↓
                 (when done, merge back into main)
```

**Real workflow:**
```
main branch         = production code (live, working)
feature/login       = you're building the login feature
feature/payments    = teammate building payments
bugfix/null-error   = fixing a bug

Everyone works in their own branch.
Nothing breaks main until it's reviewed and merged.
```

### 5.1 Branch Commands
```bash
# See all branches
git branch

# Output:
# * main          ← asterisk = current branch
#   feature/login
#   bugfix/header

# Create a new branch
git branch feature/user-login

# Switch to a branch
git checkout feature/user-login
# Modern way (Git 2.23+):
git switch feature/user-login

# Create AND switch in one command (most common)
git checkout -b feature/user-login
# Modern way:
git switch -c feature/user-login

# Delete a branch (after merging)
git branch -d feature/user-login

# Force delete (even if not merged)
git branch -D feature/user-login

# See branches with last commit
git branch -v
```

### 5.2 Merging Branches
```bash
# Step 1: Switch to the branch you want to merge INTO
git switch main

# Step 2: Merge the feature branch
git merge feature/user-login

# Git output (fast-forward):
# Updating f9a1d23..a3f8d91
# Fast-forward
#  main.py | 15 +++++++++++++++
#  1 file changed, 15 insertions(+)
```

**Types of merges:**
```
Fast-forward merge (simple):
  main:    A → B
  feature:      → C → D
  result:  A → B → C → D
  (just moves main forward, no extra commit)

3-way merge (when main also moved forward):
  main:    A → B → C
  feature:      → D → E
  result:  A → B → C → merge commit
                  ↗
                 D → E
  (creates a new "merge commit")
```

### 5.3 Merge Conflicts
A conflict happens when two branches changed the **same line** differently.

**Git tells you:**
- Auto-merging main.py
- CONFLICT (content): Merge conflict in main.py
- Automatic merge failed; fix conflicts and then commit the result.

**Open the conflicted file. You'll see:**
```python
# main.py
def get_user():
<<<<<<< HEAD
    return {"user": "from main branch"}
=======
    return {"user": "from feature branch"}
>>>>>>> feature/user-login
```

**Reading the conflict:**
```
<<<<<<< HEAD
    code from YOUR current branch (main)
=======
    code from the branch you're merging in
>>>>>>> feature/user-login
```

**Resolving it:**
```python
# Delete the conflict markers, keep what you want:
def get_user():
    return {"user": "from feature branch"}  # chose this one
```

```bash
# Then:
git add main.py
git commit -m "Merge feature/user-login into main"
```

**VS Code makes this easier:**
```
VS Code shows conflict files with colored highlights
and buttons:
  "Accept Current Change"    ← keep HEAD version
  "Accept Incoming Change"   ← keep feature version
  "Accept Both Changes"      ← keep both
  "Compare Changes"          ← see side by side
```

---
## 6. GitHub - Remote Repositories

### 6.1 Local vs Remote
```
LOCAL repository:
  Lives on YOUR computer
  Only you can see it
  Lost if your laptop dies

REMOTE repository (GitHub):
  Lives on GitHub's servers
  Team can access it
  Safe backup forever
  Foundation for collaboration
```

### 6.2 Setting Up GitHub
1. Go to [github.com](https://github.com)
2. Sign up for a free account
3. Verify your email
4. Set up your profile:
   - Add a profile photo
   - Add your name
   - Add a bio: "Python Backend Developer"

### 6.3 SSH Keys - Authenticate Securely
Instead of typing your password every time, SSH keys let GitHub know it's really you.

**Step 1: Generate SSH key**
```bash
# Generate a new SSH key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Press Enter for default location
# Enter a passphrase (optional but recommended)

# Your keys are now at:
# ~/.ssh/id_ed25519      ← private key (NEVER share this)
# ~/.ssh/id_ed25519.pub  ← public key (share this with GitHub)
```

**Step 2: Copy your public key**
```bash
# Mac:
cat ~/.ssh/id_ed25519.pub | pbcopy

# Linux:
cat ~/.ssh/id_ed25519.pub | xclip -selection clipboard

# Windows:
cat ~/.ssh/id_ed25519.pub | clip

# Or just print and manually copy:
cat ~/.ssh/id_ed25519.pub
```

**Step 3: Add to GitHub**
1. Go to [github.com](https://github.com)
2. Click your profile photo → Settings
3. Left sidebar → SSH and GPG keys
4. Click "New SSH key"
5. Title: "My Laptop" (or whatever device this is)
6. Paste your public key
7. Click "Add SSH key"

**Step 4: Test connection**
```bash
ssh -T git@github.com
# Output: Hi username! You've successfully authenticated.
```

### 6.4 Pushing to GitHub
```bash
# Step 1: Create a new repo on github.com
# (Click the "+" button → New repository)
# Name it, make it public or private, click Create

# Step 2: Connect your local repo to GitHub
git remote add origin git@github.com:yourusername/my_project.git

# "origin" = nickname for your GitHub repo URL
# You can verify:
git remote -v
# Output:
# origin  git@github.com:yourusername/my_project.git (fetch)
# origin  git@github.com:yourusername/my_project.git (push)

# Step 3: Push your code
git push -u origin main

# -u = set upstream (only needed first time)
# After that, just:
git push
```

### 6.5 Pulling from GitHub
```bash
# Get latest changes from GitHub
git pull

# This does two things:
# 1. git fetch  (download changes)
# 2. git merge  (merge into your branch)

# Or do them separately:
git fetch origin
git merge origin/main

# Pull a specific branch
git pull origin feature/payments
```

### 6.6 The Complete Team Workflow
```bash
# Morning: get latest code
git pull

# Start a new feature
git switch -c feature/add-search

# Work, work, work...
# Make changes to files

# Save progress
git add .
git commit -m "Add search endpoint with filters"

# More work...
git add .
git commit -m "Add tests for search endpoint"

# Push to GitHub
git push -u origin feature/add-search

# Go to GitHub and open a Pull Request
# Teammate reviews your code
# After approval → merge into main
# Delete the feature branch

# Back to main and pull the merged code
git switch main
git pull
```

---
## 7. `.gitignore` - What Git Should Ignore

Some files should NEVER go to GitHub:
```python
# .gitignore

# Virtual environment
venv/
.venv/
env/

# Environment variables (SECRETS)
.env
.env.local
.env.production

# Python cache files
__pycache__/
*.pyc
*.pyo
*.pyd
.Python

# Testing
.pytest_cache/
.coverage
htmlcov/

# IDE files
.vscode/settings.json
.idea/
*.swp
*.swo

# OS files
.DS_Store          # Mac
Thumbs.db          # Windows

# Build files
dist/
build/
*.egg-info/

# Logs
*.log
logs/

# Database files
*.db
*.sqlite3
```

**Create this file in your project root:**
```bash
touch .gitignore
# Add the contents above
git add .gitignore
git commit -m "Add .gitignore"
```

**GitHub also has templates:**
```
When creating a repo on GitHub:
  "Add .gitignore" → select "Python"
  GitHub auto-generates a good Python .gitignore
```

---
## 8. Pull Requests & Code Reviews
A Pull Request is NOT a Git feature. It's a GitHub feature.
```
It says:
"I've finished my feature branch.
 Please review my code.
 If it looks good, merge it into main."
```
This is how ALL professional teams work. Nobody pushes directly to main.

### 8.1 Creating a Pull Request
1. Push your feature branch to GitHub - `git push -u origin feature/user-login`
2. Go to [github.com](https://github.com) → your repository
3. GitHub shows a banner: "feature/user-login had recent pushes" - Click **"Compare & pull request"**
4. Fill in:
   - **Title:** "Add user login endpoint"
   - **Description:** What you did, why, how to test
5. Click **"Create pull request"**
6. Teammate reviews:
   - Reads your code
   - Leaves comments
   - Requests changes OR approves
7. After approval:
   - Click **"Merge pull request"**
   - Delete the branch

### 8.2 Writing Good PR Descriptions
```
## What this PR does
Adds JWT-based authentication for user login.
Users can now POST to /auth/login with email/password
and receive an access token.

## Changes made
- Added /auth/login endpoint
- Added JWT token generation
- Added password hashing with bcrypt
- Added tests for login flow

## How to test
1. Start the server: uvicorn main:app
2. POST to http://localhost:8000/auth/login
3. Body: {"email": "test@test.com", "password": "password123"}
4. Should receive: {"access_token": "...", "token_type": "bearer"}

## Related issues
Closes #42
```

---
## 9. Git Flow & Conventional Commits

### 9.1 Branch Naming Conventions
```
feature/what-you-built     → new functionality
bugfix/what-you-fixed      → bug fixes
hotfix/critical-fix        → urgent production fix
release/version-number     → preparing a release
chore/what-you-did         → maintenance tasks

Examples:
  feature/user-authentication
  feature/payment-integration
  bugfix/null-pointer-login
  hotfix/security-vulnerability
  chore/update-dependencies
```

### 9.2 Conventional Commits
Professional teams use a standard commit message format:
```
type(scope): short description

[optional body]

[optional footer]
```

**Types:**

| Type | Meaning |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code restructure, no feature/fix |
| `test` | Adding or updating tests |
| `chore` | Maintenance, dependency updates |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |

**Examples:**
```bash
git commit -m "feat(auth): add JWT refresh token endpoint"
git commit -m "fix(users): handle null email in registration"
git commit -m "docs(api): update authentication documentation"
git commit -m "test(auth): add tests for token expiration"
git commit -m "refactor(database): move queries to repository layer"
git commit -m "chore(deps): upgrade FastAPI to 0.104.1"
git commit -m "perf(search): add database index for search queries"
```

**Why this matters:**
- Auto-generates changelogs
- Makes git log readable
- Everyone on team follows same format
- Required by many companies
- Shows professionalism

---
## 10. Practical Exercise - The Dev Journal Project
This is your Phase 0 mini project from the roadmap.

```bash
# 1. Create the project
mkdir dev-journal
cd dev-journal

# 2. Initialize Git
git init

# 3. Create the structure
mkdir entries notes resources

# 4. Create files
touch README.md
touch .gitignore
touch entries/2024-01-15.md

# 5. Add .gitignore content
echo ".DS_Store
Thumbs.db
*.log
.env" > .gitignore
```

```
<!-- README.md -->
# Developer Journal

My daily learning journal as I become a Python backend engineer.

## Structure
- entries/ — daily learning entries
- notes/ — topic-specific notes
- resources/ — useful links and references

## Entry Format
Each entry covers:
- What I learned today
- Code I wrote
- Problems I solved
- Questions I still have
```

```
<!-- entries/2024-01-15.md -->
# January 15, 2024

## What I Learned
- How the internet works (DNS, HTTP, TCP/IP)
- Set up my Python development environment
- Learned Git basics

## Code I Wrote
- Set up my first virtual environment
- Made first API call with requests library

## Problems I Solved
- Figured out how to activate venv on my OS

## Questions I Still Have
- How exactly does TLS encryption work internally?
- What's the difference between WSGI and ASGI?
```

```bash
# 6. Make your first commit
git add .
git commit -m "feat: initialize dev journal repository"

# 7. Create a new branch
git switch -c feature/add-git-notes

# 8. Create a notes file
touch notes/git-commands.md
```

```
<!-- notes/git-commands.md -->
# Git Commands Reference

## Daily Commands
- git status — check what's changed
- git add . — stage all changes
- git commit -m "message" — save snapshot
- git push — send to GitHub
- git pull — get latest from GitHub

## Branching
- git switch -c branch-name — create and switch
- git switch main — go back to main
- git merge branch-name — merge branch
```

```bash
# 9. Commit on the branch
git add .
git commit -m "docs: add git commands reference notes"

# 10. Merge back to main
git switch main
git merge feature/add-git-notes

# 11. Delete the branch
git branch -d feature/add-git-notes

# 12. View your history
git log --oneline --graph

# 13. Push to GitHub
# (Create repo on GitHub first, then:)
git remote add origin git@github.com:yourusername/dev-journal.git
git push -u origin main
```
**Keep this journal updated as you learn. Every topic = an entry.**

---
## Visual Summary - Complete Git Picture
```
┌──────────────────────────────────────────────────────────────┐
│                    GIT WORKFLOW                              │
│                                                              │
│  YOUR COMPUTER                    GITHUB                     │
│  ─────────────                    ──────                     │
│                                                              │
│  Working Directory                                           │
│  (editing files)                                             │
│       │                                                      │
│    git add                                                   │
│       │                                                      │
│  Staging Area                                                │
│  (ready to commit)                                           │
│       │                                                      │
│    git commit                                                │
│       │                                                      │
│  Local Repository  ──git push──►  Remote Repository          │
│  (your history)    ◄─git pull──   (GitHub)                   │
│                                                              │
│  BRANCHING:                                                  │
│                                                              │
│  main    ●──────────────────────────●  (stable)              │
│           \                        /                         │
│  feature   ●────●────●────●────●──  (your work)              │
└──────────────────────────────────────────────────────────────┘  ┌──────────────────────────────────────────────────────────────┐  │  KEY COMMANDS:                                               │
│  git init          start tracking                            │
│  git clone         download from GitHub                      │
│  git status        see what changed                          │
│  git add           stage changes                             │
│  git commit        save snapshot                             │
│  git push          send to GitHub                            │
│  git pull          get from GitHub                           │
│  git branch        manage branches                           │
│  git switch        change branches                           │
│  git merge         combine branches                          │
│  git log           view history                              │
└──────────────────────────────────────────────────────────────┘
```

---
## Knowledge Check
1. What is the difference between `git add` and `git commit`?
2. What does `git status` show you?
3. What is a branch and why do we use them?
4. What is a merge conflict and how do you resolve it?
5. What is the difference between `git pull` and `git fetch`?
6. What goes in `.gitignore` and why?
7. What is a Pull Request?
8. What does `git log --oneline` show?
9. How do you undo a commit but keep the changes?
10. What is the conventional commit format?
11. Why should you never commit the `.env` file?
12. What is the difference between `origin` and `main`?

---
## How This Connects to Your Future Work
As a Python backend engineer, every single day you will:
- git pull (start of day - get latest code)
- git switch -c (start a new feature/fix)
- git add + commit (save your progress constantly)
- git push (share your work)
- Open Pull Requests (get code reviewed)
- Review teammates' PRs (give feedback)
- Resolve conflicts (when branches collide)

Git is not optional. It's as fundamental as knowing Python. Every job listing says "Git experience required". Now you have it.

---