Before a carpenter builds furniture, they set up their workshop.  
Before you write Python, you set up your environment.

**A *development environment* is:**
```
Your computer
    │
    ├── Python (the language interpreter)
    ├── VS Code (where you write code)
    ├── Terminal (how you control everything)
    ├── Virtual Environments (isolated workspaces per project)
    └── pip / Poetry (how you install tools)
```
**Get this right once. It'll serve you for years.**

---
## 1. Installing Python

### 1.1 What is Python the interpreter?
```
You write:          Python reads it:         Computer does it:
─────────────────────────────────────────────────────────────
print("Hello")  →  "oh, print means         → displays
                    show text on screen"       "Hello"
```
Python is the program that READS and RUNS your code. Without it, your `.py` files are just text files.

### 1.2 Installing Python
**Step 1: Check if Python is already installed**
Open your terminal and type:
```bash
python3 --version
```

You'll see one of these:
```bash
# Good — already installed:
Python 3.12.3

# Not installed:
command not found: python3
```

**Step 2: Install Python 3.12+**
**Windows:**
1. Go to [python.org/downloads](https://python.org/downloads)
2. Download Python 3.12.x (the latest 3.12)
3. Run the installer
4. ⚠️ **CRITICAL:** Check **"Add Python to PATH"** checkbox.
5. Click "Install Now"
6. Verify: Open Command Prompt → type: `python --version`

**Mac:**
```bash
# Option 1: Using Homebrew (recommended)
# First install Homebrew if you don't have it:
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Then install Python:
brew install python@3.12

# Verify:
python3 --version
```

**Linux (Ubuntu/Debian):**
```bash
# Update package list first
sudo apt update

# Install Python
sudo apt install python3.12 python3.12-venv python3-pip

# Verify
python3 --version
```

### 1.3 `python` vs `python3`
```
On most systems:
  python   → might mean Python 2 (OLD, avoid)
  python3  → definitely Python 3 (what we want)

Always use python3 to be safe.
Or set up an alias (we'll handle this automatically with venv).
```

### 1.4 What NOT to do?
```
❌ Don't install multiple Python versions randomly
❌ Don't use system Python for your projects
   (system Python = the one your OS uses internally)
❌ Don't install packages globally with pip

✅ Always use virtual environments (coming up)
✅ Keep Python version consistent per project
```

---
## 2. VS Code - Your Code Editor
Visual Studio Code (commonly known as VS Code) is a free, lightweight, and highly popular code editor developed by Microsoft.
### 2.1 Why VS Code?
Options you might have heard of:
```
┌─────────────────┬──────────────────────────────────────┐
│ Editor          │ Verdict                              │
├─────────────────┼──────────────────────────────────────┤
│ VS Code         │  Best for beginners + professionals  │
│ PyCharm         │  Great but heavy, paid for full ver  │
│ Vim/NeoVim      │  Powerful but steep learning curve   │
│ Jupyter         │  Good for data science, not backend  │
│ Notepad         │  Please don't                        │
└─────────────────┴──────────────────────────────────────┘
```
We use VS Code. It's free, fast, and used by millions.

### 2.2 Installing VS Code
1. Go to [code.visualstudio.com](https://code.visualstudio.com)
2. Download for your OS
3. Install it
4. Open it

### 2.3 Essential Extensions
Open VS Code. Press `Ctrl+Shift+X` (Windows/Linux) or `Cmd+Shift+X` (Mac). This opens the Extensions panel. Search and install each one:

**1. Python (by Microsoft)**
```
What it does:
  - Lets VS Code understand Python code
  - Syntax highlighting (colors your code)
  - Runs Python files
  - IntelliSense (auto-complete as you type)

Search: "Python"
Install: The one by Microsoft (millions of downloads)
```

**2. Pylance (by Microsoft)**
```
What it does:
  - Super-powered Python language support
  - Better auto-complete
  - Type checking as you type
  - Shows you what functions expect as input

Usually auto-installs with Python extension.
If not, search "Pylance".
```

**3. GitLens**
```
What it does:
  - Shows who wrote each line of code
  - Git history inside VS Code
  - Blame annotations
  - Compare file versions

Search: "GitLens"
```

**4. Thunder Client**
```
What it does:
  - Test your APIs without leaving VS Code
  - Like Postman but built into VS Code
  - Send HTTP requests, see responses

Search: "Thunder Client"
```

**5. (Bonus) These are also great:**
```
- "indent-rainbow"    → colorizes indentation (Python needs this)
- "Error Lens"        → shows errors inline on the same line
- "Auto Rename Tag"   → for HTML (you'll occasionally touch it)
- "Prettier"          → code formatting (more for JS but useful)
```

### 2.4 VS Code Settings
Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac). Type "Open Settings JSON". Select it. Add these settings:

```json
{
  "editor.fontSize": 14,
  "editor.tabSize": 4,
  "editor.insertSpaces": true,
  "editor.formatOnSave": true,
  "editor.rulers": [88],
  "python.defaultInterpreterPath": "python3",
  "editor.minimap.enabled": false,
  "files.autoSave": "afterDelay",
  "editor.wordWrap": "on",
  "terminal.integrated.fontSize": 13
}
```

**What these do:**
```
tabSize: 4          → Python standard is 4 spaces per indent
formatOnSave: true  → Auto-format code when you save
rulers: [88]        → Shows a line at 88 chars (Black formatter standard)
autoSave            → Saves your file automatically
```

### 2.5 VS Code Keyboard Shortcuts

| Action                 | Windows/Linux | Mac            |
| ---------------------- | ------------- | -------------- |
| Open terminal          | Ctrl+\`       | Cmd+\`         |
| Open file              | Ctrl+P        | Cmd+P          |
| Command palette        | Ctrl+Shift+P  | Cmd+Shift+P    |
| Save file              | Ctrl+S        | Cmd+S          |
| Find in file           | Ctrl+F        | Cmd+F          |
| Find in ALL files      | Ctrl+Shift+F  | Cmd+Shift+F    |
| Comment/uncomment line | Ctrl+/        | Cmd+/          |
| Duplicate line         | Shift+Alt+↓   | Shift+Option+↓ |
| Move line up/down      | Alt+↑/↓       | Option+↑/↓     |
| Multiple cursors       | Ctrl+Click    | Cmd+Click      |
| Select all occurrences | Ctrl+Shift+L  | Cmd+Shift+L    |
| Format document        | Shift+Alt+F   | Shift+Option+F |
| Go to definition       | F12           | F12            |
| Run Python file        | F5            | F5             |
| Split editor           | Ctrl+\        | Cmd+\          |

**Learn these. They will save you hours every week.**

---
## 3. The Terminal
The terminal is a text-based way to control your computer.
```
Instead of:                    You type:
Click folder → click file  →   cd documents/myproject
Double click program       →   python3 main.py
Create new folder          →   mkdir new_folder
```
As a backend engineer, you'll live in the terminal. Servers don't have graphical interfaces. Just terminal. use [[Terminal Cheatsheet]] for quick access.

### 1.1 Opening the Terminal
```
Windows:
  - Search "Command Prompt" or "PowerShell"
  - Or install Windows Terminal (better, from Microsoft Store)
  - Or use Git Bash (installs with Git)
  - Best: Use VS Code's built-in terminal (Ctrl+`)
```

```
Mac:
  - Search "Terminal" in Spotlight (Cmd+Space)
  - Or install iTerm2 (better terminal app)
  - Or use VS Code's built-in terminal (Cmd+`)
```

```
Linux:
  - Ctrl+Alt+T on most distros
  - Or use VS Code's built-in terminal
```

### 3.2 Essential Terminal Commands

#### Navigation
```bash
# Where am I right now?
pwd
# Output: /home/yourname/documents

# List files and folders here
ls
# Output: projects/  downloads/  notes.txt

# List with details
ls -la

# Go into a folder
cd projects

# Go back one level
cd ..

# Go back two levels
cd ../..

# Go to home directory
cd ~

# Go to exact path
cd /home/yourname/documents/myproject
```

#### Files & Folders
```bash
# Create a folder
mkdir my_project

# Create a file
touch main.py

# Create nested folders
mkdir -p projects/backend/myapp

# Copy a file
cp main.py main_backup.py

# Move/rename a file
mv main.py app.py

# Delete a file (careful — no trash, permanent)
rm main.py

# Delete a folder and everything in it (BE CAREFUL)
rm -rf my_folder/

# Print file contents
cat main.py

# Print first 10 lines
head main.py

# Print last 10 lines
tail main.py
```

#### Running Things
```bash
# Run a Python file
python3 main.py

# Install a package
pip install requests

# See installed packages
pip list

# Clear the terminal screen
clear
```

### 3.3 Terminal Inside VS Code
You don't need to switch between VS Code and Terminal. VS Code has a built-in terminal.

```
Open it:  Ctrl+` (backtick — the key above Tab)

You can have MULTIPLE terminals open at once.
Click the "+" button in the terminal panel.

This is what you'll use 90% of the time.
```

---
## 4. Virtual Environments
The Most Important Concept Here.
### 4.1 The Problem Without VE
Imagine this:
```
You have 2 projects:

Project A (built in 2022):          Project B (built in 2024):
  Uses Django version 3.2              Uses Django version 5.0
  Uses requests version 2.26           Uses requests version 2.31

These versions are INCOMPATIBLE.
If you install both, they fight each other.
```

**Without virtual environments:**
```
Your computer (global Python)
  ├── Django 3.2  ← Project A needs this
  ├── Django 5.0  ← Project B needs this
  ├── ??? CONFLICT — only one can exist
  └── Everything breaks
```

**With virtual environments:**
```
Your computer
  ├── project_a_env/          ← isolated bubble
  │     ├── Django 3.2        ← only for project A
  │     └── requests 2.26
  │
  └── project_b_env/          ← isolated bubble
        ├── Django 5.0        ← only for project B
        └── requests 2.31

No conflicts. Each project lives in its own world.
```

**Think of virtual environments as:**
- Each project gets its own clean apartment.
- They don't share furniture (packages).
- They don't interfere with each other.

### 4.2 Creating Your First VE
**Step 1: Create a project folder**
```bash
# Go to where you keep projects
cd ~
mkdir projects
cd projects

# Create your first project folder
mkdir my_first_project
cd my_first_project
```

**Step 2: Create a virtual environment**
```bash
# This creates a folder called "venv" inside your project
python3 -m venv venv #mostly use env

# What this does:
# python3         → use Python 3
# -m venv         → run the venv module
# venv            → name of the environment folder
```

You'll see a new folder called `env/` appear:
```python
my_first_project/
└── venv/
    ├── bin/          #( Mac/Linux) - contains Python, pip
    ├── Scripts/      # (Windows) - contains Python, pip
    ├── lib/          # where packages get installed
    └── pyvenv.cfg    # config file
```

**Step 3: Activate the virtual environment**
```bash
# Mac/Linux:
source venv/bin/activate

# Windows (Command Prompt):
venv\Scripts\activate.bat

# Windows (PowerShell):
venv\Scripts\Activate.ps1
```

**After activation, your terminal changes:**
```bash
# Before activation:
yourname@computer:~/projects/my_first_project$

# After activation:
(env) yourname@computer:~/projects/my_first_project$
  ↑
# This (venv) prefix tells you it's active
```

**Step 4: Verify it's using the right Python**
```bash
which python3
# Should show: /home/yourname/projects/my_first_project/venv/bin/python3
# NOT the system Python
```

**Step 5: Deactivate when done**
```bash
deactivate

# Prompt goes back to normal (no venv prefix)
```

### 4.3 The Golden Rules of VE
```
Rule 1: ONE virtual environment per project
        Never share environments between projects

Rule 2: ALWAYS activate before working
        Check for (venv) in your prompt

Rule 3: NEVER commit the venv/ folder to Git
        Add it to .gitignore (we'll cover this)

Rule 4: Install packages ONLY when activated
        If you don't see (venv), don't install

Rule 5: Use venv/ as the folder name
        It's the convention everyone follows
```

---
## 5. pip & Package Management
pip = Python's package installer, like an app store, but for Python libraries.

Instead of manually downloading code, you just say: pip install requests. And pip downloads and installs it automatically.

### 5.1 Basic pip Commands
```bash
# Always activate your venv first!
source venv/bin/activate

# Install a package
pip install requests

# Install specific version
pip install requests==2.31.0

# Install minimum version
pip install requests>=2.28.0

# Upgrade a package
pip install --upgrade requests

# Uninstall a package
pip uninstall requests

# List all installed packages
pip list

# Show info about a package
pip show requests

# Search for packages (go to pypi.org instead, it's better)
pip search requests
```

### 5.2 `requirements.txt`
Sharing Your Dependencies.

**The problem:**
```
You build a project.
Your friend wants to run it.
How do they know which packages to install?
```

**The solution:** `requirements.txt`
```bash
# Generate requirements.txt from your current environment
pip freeze > requirements.txt
```

**This creates a file like:**
```java
# requirements.txt
requests==2.31.0
fastapi==0.104.1
sqlalchemy==2.0.23
pydantic==2.5.0
uvicorn==0.24.0
```

**Your friend (or server) installs everything with one command:**
```bash
pip install -r requirements.txt
```

### 5.3 `.env` Files
Keeping Secrets Safe.

**Your app will have secrets:**
```
Database password: "super_secret_password"
API key: "sk-abc123..."
Secret key: "random_string_for_jwt"
```

**NEVER put these directly in your code:**
```python
# ❌ WRONG — never do ths
DATABASE_URL = "postgresql://admin:super_secret_password@localhost/mydb"
API_KEY = "sk-abc123..."

# If you push this to GitHub, the whole world sees your secrets
```

**Use `.env` files instead:**
Create a file called `.env` in your project root:
```bash
# .env file
DATABASE_URL=postgresql://admin:super_secret_password@localhost/mydb
API_KEY=sk-abc123...
SECRET_KEY=my_random_jwt_secret
DEBUG=True
```

**Install `python-dotenv`:**
```bash
pip install python-dotenv
```

**Load it in your Python code:**
```python
# main.py
from dotenv import load_dotenv
import os

load_dotenv()  # reads .env file

database_url = os.getenv("DATABASE_URL")
api_key = os.getenv("API_KEY")

print(database_url)  # postgresql://admin:super_secret_password@...
```

**Add `.env` to `.gitignore` so it never gets committed:**
```bash
# .gitignore
venv/
.env
```

Watch [this](https://youtu.be/Y21OR1OPC9A) if you don't understand.

---
## 6. Poetry
Modern Dependency Management.
### 6.1 Why Poetry When We Have pip?
pip + `requirements.txt` problems:
```
  ❌ No separation of dev vs production dependencies
  ❌ requirements.txt can get out of sync
  ❌ No dependency conflict resolution
  ❌ No built-in virtual env management
  ❌ Versions can be loose/unpredictable
```
Poetry solves ALL of this. Industry is moving toward Poetry. Many companies use it.

Poetry advantages:
```
  ✅ Automatic virtual environment management
  ✅ Separates dev and production dependencies
  ✅ Lock file (exact reproducible installs)
  ✅ Dependency conflict detection
  ✅ Clean project structure (pyproject.toml)
  ✅ One tool to rule them all
```

**Note:** Poetry must be installed globally.
### 6.2 Installing Poetry
```bash
# Mac/Linux:
curl -sSL https://install.python-poetry.org | python3 -

# Apply the Changes - Mac
source ~/.zshrc
```

```bash
# Windows (PowerShell):
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python -
```

### 6.3 The Recommended Alternative: Using `Pipx` (Mac)
The official documentation prefers using `pipx` to manage standalone Python applications like Poetry. It isolates the package in its own environment while still utilizing Homebrew safely.

Install `pipx` via `Homebrew`:
```bash
brew install pipx
```

Configure your shell PATH variables:
```bash
pipx ensurepath
```
Restart your terminal window.

Install Poetry via pipx:
```bash
pipx install poetry
```

```bash
# Verify installation:
poetry --version
# Output: Poetry (version 1.7.x)
```

### 6.4 Using Poetry

**Start a new project:**
```bash
# Create new project
poetry new my_backend_project
```

**OR use `poetry` in an existing project:**
```bash
cd my_existing_project
poetry init
# Poetry asks you questions, creates pyproject.toml
```

**The structure `poetry` creates:**
```bash
# Structure created:
my_backend_project/
├── pyproject.toml   ← the main config file (replaces requirements.txt)
├── README.md
├── my_backend_project/
│   └── __init__.py
└── tests/
    └── __init__.py
```

**The `pyproject.toml` file:**
The file in **Poetry** serves as the single, centralized configuration file used to **manage project dependencies, declare metadata, and configure build environments**. It replaces legacy configuration files like `setup.py`, `requirements.txt`, and `setup.cfg` into a standardized, modern format.

```toml
[tool.poetry]
name = "my-backend-project"
version = "0.1.0"
description = "My first backend project"
authors = ["Your Name <you@email.com>"]

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.104.1"
sqlalchemy = "^2.0.23"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
black = "^23.0.0"
ruff = "^0.1.0"
```

**Highly Recommended Mac Optimization:**
By default, [Poetry Documentation](https://python-poetry.org/docs/configuration/) states that environments are created globally in your system's cache folder (`~/Library/Caches/pypoetry`).

To keep your project clean and make it easier for code editors like **VS Code** or **PyCharm** to find your Python interpreter, run this command **before** creating your environment:
```bash
poetry config virtualenvs.in-project true
```
This forces Poetry to build the environment right inside your project directory in a hidden folder called `.venv`.

Or you can just create a virtual environment with `.env` method.

**Installing packages with Poetry:**
```bash
# Add a production dependency
poetry add fastapi

# Add with specific version
poetry add fastapi@0.104.1

# Add a dev-only dependency (testing, linting tools)
poetry add --group dev pytest

# Remove a package
poetry remove requests
```

```bash
# Install all dependencies (like pip install -r requirements.txt)
poetry install

# Update all packages
poetry update
```

```bash
# Activate Poetry's virtual environment
source <(poetry env activate)
# Now you're inside the venv
```

**Running your code with Poetry:**
```bash
# Run Python
python main.py # copy the code below

# OR run without activating
poetry run python main.py
poetry run pytest
```

```python title:main.py
print("Hello World!") # copy this to your main.py
```

**The lock file:**
```bash
# Poetry creates poetry.lock automatically
# This file locks EXACT versions of all packages
# Commit this to Git — teammates get identical environments

my_project/
├── poetry.toml.      ← commit this
├── pyproject.toml    ← commit this
└── poetry.lock       ← commit this
```

### 6.5 pip vs Poetry
| Feature                    | Pip                                          | Poetry                                                 |
| -------------------------- | -------------------------------------------- | ------------------------------------------------------ |
| **Primary Role**           | Package Installer                            | Project & Dependency Manager                           |
| **Virtual Environments**   | None; requires external tools (e.g., `venv`) | Automatic; creates and manages them for you            |
| **Dependency Resolution**  | Minimal; can lead to version conflicts       | Advanced; auto-checks compatibility and locks versions |
| **Configuration File**     | Plain text `requirements.txt`                | Structured `pyproject.toml`                            |
| **Packaging & Publishing** | Not supported directly                       | Supported out-of-the-box                               |

When to Use Which?
```
Use pip + venv when:
  - Learning/personal projects
  - Quick scripts
  - The project already uses requirements.txt
  - Working on a team that uses pip
```

```
Use Poetry when:
  - New professional projects
  - Team projects (ensures everyone has same deps)
  - When you need dev/prod dependency separation
  - When you want modern tooling
```
**Reality:** You'll encounter BOTH in jobs. Know both. We'll primarily use Poetry going forward.

---
## 7. How to Read Official Documentation?
This Skill is More Valuable Than Any Tutorial.
Tutorials get outdated. Documentation is always current.
Senior engineers read docs constantly. Junior engineers avoid docs and get stuck.
The difference in your growth speed is massive.

### 7.1 The Main Docs You'll Visit Constantly

| Technology | Documentation URL |
|-----------|-----------------|
| Python | [docs.python.org/3/](https://docs.python.org/3/) |
| FastAPI | [fastapi.tiangolo.com](https://fastapi.tiangolo.com) |
| Django | [docs.djangoproject.com](https://docs.djangoproject.com) |
| SQLAlchemy | [docs.sqlalchemy.org](https://docs.sqlalchemy.org) |
| Pydantic | [docs.pydantic.dev](https://docs.pydantic.dev) |
| pytest | [docs.pytest.org](https://docs.pytest.org) |
| Docker | [docs.docker.com](https://docs.docker.com) |
| PostgreSQL | [postgresql.org/docs/](https://postgresql.org/docs/) |
| Redis | [redis.io/docs/](https://redis.io/docs/) |

---
### 7.2 How to Read Documentation Effectively
```
Step 1: Start with "Getting Started" or "Quickstart"
        Get something running fast.
        Don't read everything first.

Step 2: Understand the structure
        Most docs have:
        - Tutorial (follow along)
        - How-to guides (specific tasks)
        - Reference (every function/class)
        - Explanation (concepts)

Step 3: Use search (Ctrl+F or the search bar)
        You rarely read docs top to bottom.
        Search for what you need.

Step 4: Read the function signature carefully
        def get_user(user_id: int, db: Session) -> User:
        This tells you: inputs, types, return value

Step 5: Look at examples
        Most good docs have code examples.
        Copy, run, modify, understand.

Step 6: Check the version
        Make sure you're reading docs for YOUR version.
        Python 3.12 docs ≠ Python 3.8 docs
```

### 7.3 Practical Exercise
1. Go to [docs.python.org/3/](https://docs.python.org/3/)
2. Find the documentation for the "list" type
3. Find what the `.append()` method does
4. Find what the `.sort()` method does and its parameters
5. Find the "string" methods documentation

**This is a skill you practice, not learn once.**

---
## Complete Project Structure
Every project you build should look like this:
```python title:project_structure.py
my_project/
│
├── .env              ← secrets (NEVER commit)
├── .env.example      ← template without real values (commit this)
├── .gitignore        ← tells git what to ignore
├── pyproject.toml    ← Poetry config
├── poetry.lock       ← locked dependencies (commit)
├── README.md         ← project description
│
├── src/                    ← your actual code
│   └── my_project/
│       ├── __init__.py
│       └── main.py
│
└── tests/                  ← your tests
    ├── __init__.py
    └── test_main.py
```

---
## Hands-On Practice
Let's actually set up your first proper project:
```bash
# 1. Create your projects directory
mkdir ~/projects
cd ~/projects

# 2. Create your first project
mkdir hello_backend
cd hello_backend

# 3. Create a virtual environment
python3 -m venv venv

# 4. Activate it
source venv/bin/activate    # Mac/Linux
# venv\Scripts\activate     # Windows

# 5. Verify (you should see (venv) in prompt)
which python3

# 6. Install your first package
pip install requests

# 7. Create your first Python file
touch main.py

# 8. Open in VS Code
code .

# 9. Write this in main.py:
```

```python
# main.py
import requests

response = requests.get("https://api.github.com")

print(f"Status Code: {response.status_code}")
print(f"Response: {response.json()}")
```

```bash
# 10. Run it
python3 main.py

# 11. Save your dependencies
pip freeze > requirements.txt

# 12. Check what's in it
cat requirements.txt

# 13. Deactivate when done
deactivate
```
**If you see a status code and JSON response, your environment works perfectly.** 

---
## Visual Summary
```
┌─────────────────────────────────────────────────────┐
│           YOUR DEVELOPMENT ENVIRONMENT              │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  VS Code (where you write code)             │    │
│  │    ├── Python extension                     │    │
│  │    ├── Pylance                              │    │
│  │    ├── GitLens                              │    │
│  │    └── Thunder Client                       │    │
│  └─────────────────────────────────────────────┘    │
│                      │                              │
│  ┌─────────────────────────────────────────────┐    │
│  │  Terminal (where you run things)            │    │
│  │    ├── Navigate folders (cd, ls, mkdir)     │    │
│  │    ├── Run Python (python3 main.py)         │    │
│  │    └── Manage packages (pip/poetry)         │    │
│  └─────────────────────────────────────────────┘    │
│                      │                              │
│  ┌─────────────────────────────────────────────┐    │
│  │  Virtual Environment (isolated bubble)      │    │
│  │    ├── Its own Python                       │    │
│  │    ├── Its own packages                     │    │
│  │    └── Separate per project                 │    │
│  └─────────────────────────────────────────────┘    │
│                      │                              │
│  ┌─────────────────────────────────────────────┐    │
│  │  Package Management                         │    │
│  │    ├── pip (basic, always available)        │    │
│  │    ├── Poetry (modern, professional)        │    │
│  │    └── requirements.txt / pyproject.toml    │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---
## Knowledge Check
1. What command creates a virtual environment?
2. How do you activate it on your OS?
3. How do you know if a virtual environment is active?
4. What command saves your dependencies to a file?
5. What command installs from that file?
6. Why should you NEVER commit the venv/ folder?
7. What is .env used for?
8. Why is Poetry better than plain pip?
9. What VS Code shortcut opens the terminal?
10. What does pip freeze do?

---
## How This Connects to Your Future Work
Every single project you build will:
- Have a virtual environment
- Use Poetry or pip for packages
- Have a .env file for secrets
- Have `.gitignore` protecting secrets
- Be opened and edited in VS Code
- Be run from the terminal

---
This phase setup is something you'll do
on day 1 of every new job, every new project,
every new server you configure.
Getting this right NOW saves you pain FOREVER.

---