You already learned WHAT virtual environments are in Development Environment Setup. This phase is about MASTERING them hands-on.
```
Why a dedicated phase?
Every single project you build from now on starts here.
Every job you get, day 1 starts here.
Every server you deploy to starts here.

If you get this wrong:
  ❌ "It works on my machine but not on the server"
  ❌ "Installing package X broke package Y"
  ❌ "My teammate can't run my code"
  ❌ Dependency conflicts that take hours to debug

If you get this right:
  ✅ Reproducible environments everywhere
  ✅ Clean separation between projects
  ✅ Team members get identical setups
  ✅ Deployment is predictable
```

Use [[Python venv Cheatsheet (macOS)|this]] : Python `venv` Cheatsheet (macOS);

---
## Setup For This Phase
```bash
# Create a workspace to practice everything
mkdir ~/projects/venv_mastery
cd ~/projects/venv_mastery
```

---
## 1. Why Virtual Environments Exist

### The Global Python Problem
```
When you install Python on your computer:

Python lives here:
  Mac/Linux:  /usr/bin/python3 or /usr/local/bin/python3
  Windows:    C:\Python312\python.exe

When you install packages WITHOUT a virtual environment:
They go into the GLOBAL site-packages:
  Mac/Linux:  /usr/local/lib/python3.12/site-packages/
  Windows:    C:\Python312\Lib\site-packages\

This is the GLOBAL Python — shared by everything.
```

**The conflict scenario:**
```
You (2022): build Project A
  Installs: Django==3.2, requests==2.26

You (2024): build Project B
  Installs: Django==5.0, requests==2.31

Problem: Only ONE version of Django can be installed globally.
Installing Django 5.0 REMOVES Django 3.2.
Now Project A breaks.

Your system also uses Python for tools.
Breaking global Python = breaking system tools.
```

**Visual:**
```
WITHOUT virtual environments:

┌─────────────────────────────────────────┐
│           GLOBAL PYTHON                 │
│  ┌────────────────────────────────┐     │
│  │       site-packages/           │     │
│  │   Django==5.0  ← conflict!     │     │
│  │   requests==2.31               │     │
│  └────────────────────────────────┘     │
│           ↑               ↑             │
│       Project B      Project A BROKEN   │
└─────────────────────────────────────────┘
```
p
```
WITH virtual environments:

┌────────────────────┐   ┌────────────────────┐
│  project_a/venv/   │   │  project_b/venv/   │
│  ┌──────────────┐  │   │  ┌──────────────┐  │
│  │ Django==3.2  │  │   │  │ Django==5.0  │  │
│  │ requests=2.26│  │   │  │ requests=2.31│  │
│  └──────────────┘  │   │  └──────────────┘  │
│                    │   │                    │
│  Project A         │   │  Project B         │
└────────────────────┘   └────────────────────┘
```

---
## 2. `venv` - Python's Built-in Tool

### Creating Environments
```bash
# Navigate to your project
cd ~/projects/venv_mastery

# Create a project
mkdir project_one
cd project_one

# Create virtual environment
# python3 -m venv <name>
python3 -m venv venv

# Convention: always name it "venv"
# Some teams use ".venv" (hidden folder)
# The name matters less than the consistency
```

**What gets created:**
```bash
# Look inside
ls -la venv/

# You'll see:
venv/
├── bin/          (Mac/Linux)
│   ├── python       → symlink to Python
│   ├── python3      → symlink to Python
│   ├── pip          → pip for THIS environment
│   ├── pip3         → pip for THIS environment
│   └── activate     → activation script
│
├── Scripts/      (Windows — instead of bin/)
│   ├── python.exe
│   ├── pip.exe
│   └── activate.bat
│
├── lib/
│   └── python3.12/
│       └── site-packages/  ← YOUR packages go here
│
├── include/      (header files for C extensions)
└── pyvenv.cfg    ← configuration file
```

```bash
# Look at pyvenv.cfg
cat venv/pyvenv.cfg

# Output:
# home = /usr/local/bin
# include-system-site-packages = false  ← isolated!
# version = 3.12.3
# prompt = venv
```

### Activating and Deactivating
```bash
# ─────────────────────────────────────────
# ACTIVATION
# ─────────────────────────────────────────

# Mac/Linux (bash/zsh):
source venv/bin/activate

# Mac/Linux (fish shell):
source venv/bin/activate.fish

# Windows (Command Prompt):
venv\Scripts\activate.bat

# Windows (PowerShell):
venv\Scripts\Activate.ps1

# ─────────────────────────────────────────
# HOW TO KNOW IT'S ACTIVE
# ─────────────────────────────────────────

# Your prompt changes:
# Before: username@computer:~/projects/project_one$
# After:  (venv) username@computer:~/projects/project_one$
#          ↑
#          This prefix confirms activation

# VERIFY you're using the right Python:
which python3   # Mac/Linux
where python    # Windows

# Should show path INSIDE your venv:
# /home/user/projects/project_one/venv/bin/python3 ✅
# NOT the system Python:
# /usr/bin/python3 ❌

# Also verify:
python3 --version  # Python 3.12.x
which pip3         # should also be inside venv/

# ─────────────────────────────────────────
# DEACTIVATION
# ─────────────────────────────────────────
deactivate

# Prompt goes back to normal (no venv prefix)
# You're back to system Python
```

### The Activation Script - What It Actually Does
```bash
# Curious what activate does? Read it:
cat venv/bin/activate

# Key parts it does:
# 1. Saves your old PATH
# 2. Prepends venv/bin to PATH
#    → When you type "python", it finds venv's python FIRST
# 3. Sets VIRTUAL_ENV environment variable
# 4. Changes your prompt to show (venv)
# 5. Creates deactivate function to undo all this

# You can verify:
echo $PATH
# (venv) .../venv/bin:/usr/local/bin:/usr/bin:...
#         ↑ venv is FIRST in PATH
```

### Specifying Python Version
```bash
# Create venv with specific Python version
python3.12 -m venv venv
python3.11 -m venv venv_311

# Verify version
source venv/bin/activate
python3 --version  # Python 3.12.x

# Create venv without pip (rare, but possible)
python3 -m venv venv --without-pip

# Create venv that can see system packages (rare)
python3 -m venv venv --system-site-packages
# Usually you DON'T want this — defeats the purpose

# Upgrade pip inside venv after creating
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip  # always do this!
```

---
## 3. pip - Deep Dive

### Installing Packages
```bash
# Always activate venv first!
source venv/bin/activate

# Install latest version
pip install requests

# Install specific version (exact)
pip install requests==2.31.0

# Install minimum version
pip install "requests>=2.28.0"

# Install version range
pip install "requests>=2.28.0,<3.0.0"

# Install multiple packages at once
pip install fastapi uvicorn sqlalchemy

# Install from a git repository
pip install git+https://github.com/user/repo.git

# Install from local directory (editable mode)
# Used when developing a library
pip install -e .

# Install with extra dependencies
pip install "fastapi[all]"       # installs fastapi + all optional deps
pip install "pydantic[email]"    # pydantic + email validation support
```

### Managing Installed Packages
```bash
# List all installed packages
pip list

# Output:
# Package         Version
# ----------      -------
# fastapi         0.104.1
# pip             24.0
# pydantic        2.5.0
# requests        2.31.0
# setuptools      69.0.0

# List with more detail
pip list -v

# Show info about a specific package
pip show requests

# Output:
# Name: requests
# Version: 2.31.0
# Summary: Python HTTP for Humans.
# Home-page: https://requests.readthedocs.io
# Author: Kenneth Reitz
# Requires: certifi, charset-normalizer, idna, urllib3
# Required-by: (packages that need requests)

# Check for outdated packages
pip list --outdated

# Upgrade a package
pip install --upgrade requests

# Upgrade pip itself
pip install --upgrade pip

# Uninstall a package
pip uninstall requests

# Uninstall without confirmation prompt
pip uninstall requests -y

# Uninstall multiple packages
pip uninstall requests sqlalchemy -y
```

### requirements.txt — The Standard Way
```bash
# ─────────────────────────────────────────
# GENERATING requirements.txt
# ─────────────────────────────────────────

# Freeze current environment (exact versions)
pip freeze > requirements.txt

# See what's in it
cat requirements.txt

# Example output:
# annotated-types==0.6.0
# anyio==4.2.0
# certifi==2024.2.2
# fastapi==0.104.1
# idna==3.6
# pydantic==2.5.0
# pydantic-core==2.14.5
# sniffio==1.3.0
# starlette==0.27.0
# uvicorn==0.24.0.post1
```

```bash
# ─────────────────────────────────────────
# INSTALLING FROM requirements.txt
# ─────────────────────────────────────────

# Create a fresh venv somewhere else
mkdir ~/projects/project_clone
cd ~/projects/project_clone
python3 -m venv venv
source venv/bin/activate

# Install from requirements.txt
pip install -r requirements.txt

# Now both environments have IDENTICAL packages
# This is how you share reproducible environments!

# ─────────────────────────────────────────
# BEST PRACTICES for requirements.txt
# ─────────────────────────────────────────

# Option 1: Pin exact versions (most reproducible)
# requirements.txt:
# fastapi==0.104.1
# pydantic==2.5.0
# uvicorn==0.24.0

# Option 2: Separate dev and production requirements
# requirements.txt (production):
# fastapi==0.104.1
# pydantic==2.5.0
# sqlalchemy==2.0.23

# requirements-dev.txt (development only):
# -r requirements.txt  ← include production deps
# pytest==7.4.3
# black==23.12.1
# ruff==0.1.9
# mypy==1.8.0

# Install dev requirements:
pip install -r requirements-dev.txt
```

---
## 4. `.env` Files - Environment Variables

### Why `.env` Files?
```
Configuration that CHANGES between environments:
  Development: DEBUG=True, DB=localhost
  Staging:     DEBUG=False, DB=staging-server
  Production:  DEBUG=False, DB=prod-server

Secrets that should NEVER be in code:
  API keys, database passwords, JWT secrets

Solution: .env files
  - Store config as key=value pairs
  - Load into environment variables
  - Different .env per environment
  - NEVER commit to Git
```

### Creating and Using .env Files
```bash
# Create .env file in project root
cd ~/projects/project_one
touch .env
```

```bash
# .env file contents:
# No spaces around =
# No quotes needed (but you can use them)
# Comments with #

# Application
APP_NAME=MyBackendApp
DEBUG=True
SECRET_KEY=super-secret-key-change-in-production-xyz123
API_VERSION=v1

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/mydb
DATABASE_POOL_SIZE=5
DATABASE_MAX_OVERFLOW=10

# Redis
REDIS_URL=redis://localhost:6379/0

# External APIs
STRIPE_API_KEY=sk_test_abc123xyz
SENDGRID_API_KEY=SG.abc123xyz
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_S3_BUCKET=my-app-bucket

# JWT
JWT_SECRET_KEY=your-jwt-secret-key-here
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# CORS
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8080

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@myapp.com
SMTP_PASSWORD=email-password-here
```

### Loading `.env` with `python-dotenv`
```bash
pip install python-dotenv
```

```python
# Basic usage
from dotenv import load_dotenv
import os

load_dotenv()  # loads .env from current directory

# Access variables
app_name = os.getenv("APP_NAME")
debug = os.getenv("DEBUG")
db_url = os.getenv("DATABASE_URL")

print(app_name)  # MyBackendApp
print(debug)     # "True" (string! not bool)
print(db_url)    # postgresql://postgres:...

# With default values (if var not in .env)
port = os.getenv("PORT", "8000")          # default "8000"
workers = int(os.getenv("WORKERS", "4"))  # convert to int

# ─────────────────────────────────────────
# CONVERTING TYPES
# ─────────────────────────────────────────
# Everything from .env is a STRING
# You must convert to the right type

debug_str = os.getenv("DEBUG", "False")
debug = debug_str.lower() == "true"  # proper bool conversion

pool_size_str = os.getenv("DATABASE_POOL_SIZE", "5")
pool_size = int(pool_size_str)

# ─────────────────────────────────────────
# LOADING FROM SPECIFIC FILE
# ─────────────────────────────────────────
load_dotenv(".env.development")
load_dotenv(".env.production")
load_dotenv("/path/to/specific/.env")

# ─────────────────────────────────────────
# OVERRIDE EXISTING ENV VARS
# ─────────────────────────────────────────
# By default, existing env vars are NOT overridden
# (system env vars take priority)
load_dotenv(override=True)  # .env overrides system env vars

# ─────────────────────────────────────────
# FIND DOTENV (searches parent directories)
# ─────────────────────────────────────────
from dotenv import find_dotenv
dotenv_path = find_dotenv()  # finds .env in current or parent dirs
load_dotenv(dotenv_path)
```

### The Config Module Pattern
```python
# config.py — centralize all configuration
# This is what you'll write in EVERY backend project

from dotenv import load_dotenv
import os
from typing import List

load_dotenv()

class Settings:
    """
    Application settings loaded from environment variables.
    Single source of truth for all configuration.
    """

    # ─────────────────────────────────────────
    # Application
    # ─────────────────────────────────────────
    APP_NAME: str = os.getenv("APP_NAME", "MyApp")
    DEBUG: bool = os.getenv("DEBUG", "False").lower() == "true"
    ENVIRONMENT: str = os.getenv("ENVIRONMENT", "development")
    SECRET_KEY: str = os.getenv("SECRET_KEY", "")
    API_VERSION: str = os.getenv("API_VERSION", "v1")

    # ─────────────────────────────────────────
    # Server
    # ─────────────────────────────────────────
    HOST: str = os.getenv("HOST", "0.0.0.0")
    PORT: int = int(os.getenv("PORT", "8000"))
    WORKERS: int = int(os.getenv("WORKERS", "1"))

    # ─────────────────────────────────────────
    # Database
    # ─────────────────────────────────────────
    DATABASE_URL: str = os.getenv(
        "DATABASE_URL",
        "postgresql://postgres:password@localhost:5432/mydb"
    )
    DATABASE_POOL_SIZE: int = int(os.getenv("DATABASE_POOL_SIZE", "5"))
    DATABASE_MAX_OVERFLOW: int = int(os.getenv("DATABASE_MAX_OVERFLOW", "10"))

    # ─────────────────────────────────────────
    # Redis
    # ─────────────────────────────────────────
    REDIS_URL: str = os.getenv("REDIS_URL", "redis://localhost:6379/0")

    # ─────────────────────────────────────────
    # JWT
    # ─────────────────────────────────────────
    JWT_SECRET_KEY: str = os.getenv("JWT_SECRET_KEY", "")
    JWT_ALGORITHM: str = os.getenv("JWT_ALGORITHM", "HS256")
    JWT_ACCESS_TOKEN_EXPIRE_MINUTES: int = int(
        os.getenv("JWT_ACCESS_TOKEN_EXPIRE_MINUTES", "30")
    )
    JWT_REFRESH_TOKEN_EXPIRE_DAYS: int = int(
        os.getenv("JWT_REFRESH_TOKEN_EXPIRE_DAYS", "7")
    )

    # ─────────────────────────────────────────
    # CORS
    # ─────────────────────────────────────────
    @property
    def ALLOWED_ORIGINS(self) -> List[str]:
        origins = os.getenv("ALLOWED_ORIGINS", "http://localhost:3000")
        return [origin.strip() for origin in origins.split(",")]

    # ─────────────────────────────────────────
    # Validation
    # ─────────────────────────────────────────
    def validate(self) -> None:
        """Raise error if critical settings are missing."""
        required = {
            "SECRET_KEY": self.SECRET_KEY,
            "JWT_SECRET_KEY": self.JWT_SECRET_KEY,
        }
        missing = [k for k, v in required.items() if not v]
        if missing:
            raise ValueError(
                f"Missing required environment variables: {missing}"
            )

    @property
    def is_production(self) -> bool:
        return self.ENVIRONMENT == "production"

    @property
    def is_development(self) -> bool:
        return self.ENVIRONMENT == "development"

# Create a single instance — used everywhere in the app
settings = Settings()

# Usage in other files:
# from config import settings
# print(settings.DATABASE_URL)
# print(settings.DEBUG)
```

### `.env.example` - What You DO Commit

```bash
# .env has real secrets — NEVER commit
# .env.example has structure but fake values — ALWAYS commit

# .env.example
APP_NAME=MyBackendApp
DEBUG=False
SECRET_KEY=generate-a-random-secret-key-here
API_VERSION=v1

DATABASE_URL=postgresql://user:password@localhost:5432/dbname
DATABASE_POOL_SIZE=5

REDIS_URL=redis://localhost:6379/0

JWT_SECRET_KEY=generate-a-random-jwt-secret-here
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30

STRIPE_API_KEY=sk_test_your_key_here
SENDGRID_API_KEY=your_key_here

ALLOWED_ORIGINS=http://localhost:3000
```

```bash
# When a new developer joins the team:
# 1. Clone the repo
# 2. Copy .env.example to .env
cp .env.example .env

# 3. Fill in real values in .env
# 4. Create venv and install dependencies
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 5. Run the project
python3 main.py
```

---

## 5. Poetry - Complete Walkthrough
Poetry is a tool for **dependency management** and **packaging** in Python. It allows you to declare the libraries your project depends on and it will manage (install/update) them for you. Poetry offers a `lockfile` to ensure repeatable installs, and can build your project for distribution.

For latest updates click [this](https://python-poetry.org/docs/).
### Why Poetry Over pip + requirements.txt?
```
pip + requirements.txt PROBLEMS:
  ❌ pip freeze captures EVERYTHING including transitive dependencies
     (you installed requests, but requirements.txt also has certifi,
      charset-normalizer, idna, urllib3 — requests' dependencies)
  ❌ Hard to separate direct vs transitive deps
  ❌ No separation of dev vs production deps
  ❌ No automatic dependency conflict resolution
  ❌ requirements.txt can drift out of sync

Poetry SOLUTIONS:
  ✅ pyproject.toml lists ONLY your direct dependencies
  ✅ poetry.lock captures exact versions of everything
  ✅ Separate [dev] dependency groups
  ✅ Automatic conflict detection
  ✅ Built-in virtual environment management
  ✅ Lock file guarantees identical installs everywhere
```

### Installing Poetry
```bash
# Official installation (recommended)
curl -sSL https://install.python-poetry.org | python3 -

# This installs Poetry OUTSIDE any venv
# (Poetry manages its own Python environment)

# After installation, add to PATH:
# Mac/Linux — add to ~/.bashrc or ~/.zshrc:
export PATH="$HOME/.local/bin:$PATH"

# Reload shell:
source ~/.bashrc  # or source ~/.zshrc

# Verify
poetry --version
# Poetry (version 1.7.1)

# Configure Poetry to create venvs inside project
# (recommended — keeps everything together)
poetry config virtualenvs.in-project true

# This creates .venv/ inside your project folder
# instead of in a global cache somewhere
```

### Starting a New Project with Poetry
```bash
cd ~/projects/venv_mastery

# Option 1: Poetry creates project structure
poetry new my_backend_project

# Structure created:
# my_backend_project/
# ├── pyproject.toml    ← main config
# ├── README.md
# ├── my_backend_project/
# │   └── __init__.py
# └── tests/
#     └── __init__.py

cd my_backend_project
```

```
# Option 2: Initialize in existing project
mkdir existing_project
cd existing_project
poetry init
# Poetry asks you questions interactively
# Press Enter to accept defaults
```

### Understanding `pyproject.toml`
```toml
# pyproject.toml — Poetry's main config file
# This is the MODERN Python project standard (PEP 518)

[tool.poetry]
name = "my-backend-project"
version = "0.1.0"
description = "A production-grade backend API"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
packages = [{include = "my_backend_project"}]

[tool.poetry.dependencies]
# Production dependencies
# These install when someone uses your package
python = "^3.12"                     # requires Python 3.12 or higher (not 4.0)
fastapi = "^0.104.1"                 # ^0.104.1 means >=0.104.1, <0.105.0
sqlalchemy = "^2.0.23"               # caret (^) = compatible version
pydantic = "^2.5.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
python-dotenv = "^1.0.0"
psycopg2-binary = "^2.9.9"
redis = "^5.0.1"
httpx = "^0.26.0"

[tool.poetry.group.dev.dependencies]
# Development-only dependencies
# NOT installed in production
pytest = "^7.4.3"
pytest-asyncio = "^0.23.2"
pytest-cov = "^4.1.0"
black = "^23.12.1"
ruff = "^0.1.9"
mypy = "^1.8.0"
pre-commit = "^3.6.0"
factory-boy = "^3.3.0"
faker = "^22.0.0"
httpx = "^0.26.0"  # for testing FastAPI

[tool.poetry.group.test.dependencies]
# Optional: separate test group
pytest = "^7.4.3"
pytest-asyncio = "^0.23.2"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# ─────────────────────────────────────────
# Tool configurations in same file
# ─────────────────────────────────────────

[tool.black]
line-length = 88
target-version = ["py312"]

[tool.ruff]
line-length = 88
target-version = "py312"
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
python_version = "3.12"
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### Poetry Daily Commands
```bash
# ─────────────────────────────────────────
# ADDING PACKAGES
# ─────────────────────────────────────────

# Add production dependency
poetry add fastapi
poetry add "fastapi==0.104.1"      # specific version
poetry add "fastapi>=0.100.0"      # minimum version

# Add with extras
poetry add "fastapi[all]"
poetry add "uvicorn[standard]"

# Add dev dependency
poetry add --group dev pytest
poetry add --group dev black ruff mypy  # multiple at once

# Add optional dependency group
poetry add --group docs mkdocs

# ─────────────────────────────────────────
# REMOVING PACKAGES
# ─────────────────────────────────────────
poetry remove requests
poetry remove --group dev black

# ─────────────────────────────────────────
# INSTALLING (from pyproject.toml)
# ─────────────────────────────────────────

# Install all dependencies (production + dev)
poetry install

# Install only production deps (for deployment)
poetry install --only main

# Install specific group
poetry install --with dev
poetry install --with test

# Install from lock file (exact versions)
# This is what you run after git clone
poetry install

# ─────────────────────────────────────────
# UPDATING PACKAGES
# ─────────────────────────────────────────

# Validates your `pyproject.toml` file for errors or missing fields.
poetry check

# ─────────────────────────────────────────
# UPDATING PACKAGES
# ─────────────────────────────────────────

# Update all packages (within version constraints)
poetry update

# Update specific package
poetry update fastapi

# ─────────────────────────────────────────
# RUNNING CODE
# ─────────────────────────────────────────

# Activate Poetry's virtual environment
source <(poetry env activate)
# Now you're inside the venv
python main.py
# Exit the shell
exit

# OR run without activating
poetry run python main.py
poetry run pytest
poetry run black .
poetry run ruff check .

# ─────────────────────────────────────────
# ENVIRONMENT INFO
# ─────────────────────────────────────────

# Show virtual environment path
poetry env info

# List all environments
poetry env list

# Show installed packages
poetry show

# Show dependency tree
poetry show --tree

# Check for issues
poetry check
```

### The Lock File
```bash
# poetry.lock is AUTO-GENERATED by Poetry
# NEVER edit it manually

# It contains:
#   - Exact versions of ALL packages
#     (direct + transitive dependencies)
#   - Hash of each package file (security!)
#   - Source information

# ─────────────────────────────────────────
# WHAT TO COMMIT TO GIT
# ─────────────────────────────────────────

# ✅ ALWAYS COMMIT:
git add pyproject.toml   # your dependencies declaration
git add poetry.lock      # exact reproducible install

# ❌ NEVER COMMIT:
# .venv/              (virtual environment folder)
# .env                (secrets)
# __pycache__/        (compiled Python)

# .gitignore for Poetry projects:
# .venv/
# .env
# __pycache__/
# *.pyc
# .pytest_cache/
# .mypy_cache/
# .ruff_cache/
# dist/
# *.egg-info/
```

### Poetry vs pip Comparison
```
TASK                      pip + venv                     Poetry
─────────────────────────────────────────────────────────────
Create env                python3 -m venv venv           poetry new proj
                          source venv/bin/activate
Install package           pip install fastapi            poetry add fastapi
Install dev package       pip install pytest             poetry add --group dev pytest
Save dependencies         pip freeze >                   automatic
                          requirements.txt               (pyproject.toml + lock)
Install from saved        pip install                    poetry install
  dependencies              -r requirements.txt
Run code                  python main.py                 poetry run python main.py
                          (after activating)
See deps tree             pip show package               poetry show --tree
Update packages           pip install --upgrade          poetry update
Remove package            pip uninstall package          poetry remove package
Separate dev/prod         manual (two files)             built-in groups
```

---

## 6. Complete Project Structure

### The Standard Structure Every Project Should Have
```
my_backend_project/
│
├── .env                  ← real secrets (NEVER commit)
├── .env.example          ← template (always commit)
├── .gitignore            ← what Git ignores
├── pyproject.toml        ← Poetry config (commit)
├── poetry.lock           ← exact deps (commit)
├── README.md             ← project documentation
│
├── src/                  ← source code
│   └── myapp/
│       ├── __init__.py
│       ├── main.py       ← app entry point
│       ├── config.py     ← settings/config
│       ├── models/       ← data models
│       ├── routers/      ← API routes
│       ├── services/     ← business logic
│       └── utils/        ← helper functions
│
├── tests/                ← test files
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_main.py
│   └── test_services/
│
├── scripts/              ← utility scripts
│   ├── seed_database.py
│   └── generate_key.py
│
└── docs/                 ← documentation
    └── api.md
```

### Setting Up a Complete Project
```bash
# ─────────────────────────────────────────
# FULL SETUP WALKTHROUGH
# ─────────────────────────────────────────

# 1. Create project with Poetry
cd ~/projects/venv_mastery
poetry new my_backend
cd my_backend

# 2. Configure virtualenvs in project
poetry config virtualenvs.in-project true

# 3. Add dependencies
poetry add fastapi "uvicorn[standard]" sqlalchemy pydantic python-dotenv
poetry add --group dev pytest pytest-asyncio black ruff mypy

# 4. Install everything
poetry install

# 5. Create .env
cat > .env << 'EOF'
APP_NAME=MyBackend
DEBUG=True
SECRET_KEY=dev-secret-key-change-in-production
DATABASE_URL=postgresql://postgres:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379/0
JWT_SECRET_KEY=dev-jwt-secret-change-in-production
EOF

# 6. Create .env.example (safe to commit)
cat > .env.example << 'EOF'
APP_NAME=MyBackend
DEBUG=False
SECRET_KEY=generate-a-random-key-here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
REDIS_URL=redis://localhost:6379/0
JWT_SECRET_KEY=generate-a-random-jwt-secret-here
EOF

# 7. Create .gitignore
cat > .gitignore << 'EOF'
# Virtual environment
.venv/
venv/

# Environment variables
.env
.env.local
.env.*.local

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
*.egg-info/
dist/
build/

# Testing
.pytest_cache/
.coverage
htmlcov/
.mypy_cache/
.ruff_cache/

# IDE
.vscode/settings.json
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Database
*.db
*.sqlite3
EOF

# 8. Initialize Git
git init
git add .
git commit -m "feat: initial project setup"

# 9. Verify everything
poetry run python -c "import fastapi; print(fastapi.__version__)"
poetry run pytest
```

---
## 7. Multiple Environments Pattern

### Dev / Staging / Production
```bash
# Real projects have multiple environments:
#   - Development (your laptop)
#   - Testing (CI/CD pipeline)
#   - Staging (production-like, for final testing)
#   - Production (live, real users)

# Each has its own .env file:

# .env.development
APP_ENV=development
DEBUG=True
DATABASE_URL=postgresql://postgres:password@localhost:5432/mydb_dev
LOG_LEVEL=DEBUG

# .env.staging
APP_ENV=staging
DEBUG=False
DATABASE_URL=postgresql://user:password@staging-server:5432/mydb_staging
LOG_LEVEL=INFO

# .env.production
APP_ENV=production
DEBUG=False
DATABASE_URL=postgresql://user:password@prod-server:5432/mydb_prod
LOG_LEVEL=WARNING
```

```python
# config.py — loading the right environment

import os
from dotenv import load_dotenv

# Determine which env file to load
environment = os.getenv("APP_ENV", "development")
env_file = f".env.{environment}"

# Try environment-specific file first
# Fall back to .env
if os.path.exists(env_file):
    load_dotenv(env_file)
    print(f"Loaded config from {env_file}")
else:
    load_dotenv()
    print("Loaded config from .env")

# Usage:
# APP_ENV=production python main.py
# APP_ENV=staging pytest
```

---
## 8. The Complete `.gitignore`
```bash
# Save this as your standard Python .gitignore

cat > .gitignore << 'EOF'
# ─────────────────────────────────────────
# Virtual Environments
# ─────────────────────────────────────────
venv/
.venv/
env/
.env_folder/
ENV/

# ─────────────────────────────────────────
# Environment Variables (SECRETS)
# ─────────────────────────────────────────
.env
.env.local
.env.development
.env.staging
.env.production
.env.*.local
!.env.example                     # ← allow .env.example (no real secrets)

# ─────────────────────────────────────────
# Python Cache
# ─────────────────────────────────────────
__pycache__/
*.py[cod]
*$py.class
*.pyc
*.pyo
*.pyd
.Python

# ─────────────────────────────────────────
# Distribution / Packaging
# ─────────────────────────────────────────
dist/
build/
*.egg-info/
*.egg
MANIFEST
.eggs/
wheels/
*.whl

# ─────────────────────────────────────────
# Testing
# ─────────────────────────────────────────
.pytest_cache/
.coverage
.coverage.*
coverage.xml
htmlcov/
nosetests.xml
*.cover
.hypothesis/

# ─────────────────────────────────────────
# Type Checking / Linting
# ─────────────────────────────────────────
.mypy_cache/
.dmypy.json
dmypy.json
.ruff_cache/
.pytype/

# ─────────────────────────────────────────
# IDE / Editor
# ─────────────────────────────────────────
.vscode/settings.json
.idea/
*.swp
*.swo
*~
.project
.pydevproject

# ─────────────────────────────────────────
# OS Files
# ─────────────────────────────────────────
.DS_Store                        # Mac
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db                        # Windows
desktop.ini

# ─────────────────────────────────────────
# Logs
# ─────────────────────────────────────────
*.log
logs/
log/

# ─────────────────────────────────────────
# Database Files
# ─────────────────────────────────────────
*.db
*.sqlite
*.sqlite3

# ─────────────────────────────────────────
# Uploaded Files / Media
# ─────────────────────────────────────────
media/
uploads/
static/collected/

# ─────────────────────────────────────────
# Documentation
# ─────────────────────────────────────────
docs/_build/
site/

# ─────────────────────────────────────────
# Celery
# ─────────────────────────────────────────
celerybeat-schedule
celerybeat.pid

# ─────────────────────────────────────────
# Docker
# ─────────────────────────────────────────
# (docker-compose.yml and Dockerfile should be committed)
# .docker/

# ─────────────────────────────────────────
# Temporary Files
# ─────────────────────────────────────────
tmp/
temp/
*.tmp
*.bak
*.backup
EOF
```

---
## 9. Useful Utilities
```python
# generate_key.py — useful utility script
# Run: poetry run python scripts/generate_key.py

import secrets
import string

def generate_secret_key(length: int = 64) -> str:
    """Generate a cryptographically secure secret key."""
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*"
    return "".join(secrets.choice(alphabet) for _ in range(length))

def generate_jwt_secret(length: int = 32) -> str:
    """Generate a JWT secret key."""
    return secrets.token_hex(length)

def generate_api_key(prefix: str = "sk") -> str:
    """Generate an API key with prefix."""
    random_part = secrets.token_urlsafe(32)
    return f"{prefix}_{random_part}"

if __name__ == "__main__":
    print("=== Generated Secrets ===")
    print(f"\nSECRET_KEY={generate_secret_key()}")
    print(f"\nJWT_SECRET_KEY={generate_jwt_secret()}")
    print(f"\nAPI_KEY={generate_api_key()}")
    print(f"\nADMIN_API_KEY={generate_api_key('admin')}")
    print("\n=== Copy these to your .env file ===")
```

```python
# check_env.py — verify your environment is set up correctly
import sys
import os
from pathlib import Path

def check_environment():
    """Verify we're running in a virtual environment."""
    # Check if venv is active
    in_venv = (
        hasattr(sys, 'real_prefix') or          # virtualenv
        (hasattr(sys, 'base_prefix') and         # venv
         sys.base_prefix != sys.prefix)
    )

    if not in_venv:
        print("❌ ERROR: Not running in a virtual environment!")
        print("   Run: source venv/bin/activate")
        print("   OR:  poetry shell")
        sys.exit(1)

    print(f"✅ Virtual environment active: {sys.prefix}")
    print(f"   Python: {sys.version}")

    # Check for .env file
    if not Path(".env").exists():
        print("⚠️  WARNING: .env file not found!")
        print("   Copy .env.example to .env and fill in values")
    else:
        print("✅ .env file found")

    # Check required environment variables
    required_vars = [
        "DATABASE_URL",
        "SECRET_KEY",
        "JWT_SECRET_KEY",
    ]
    missing = [var for var in required_vars if not os.getenv(var)]
    if missing:
        print(f"⚠️  WARNING: Missing environment variables: {missing}")
    else:
        print("✅ All required environment variables set")

if __name__ == "__main__":
    check_environment()
```

---
## 10. Cheat Sheet - Everything in One Place
```
─────────────────────────────────────────────────────────────
VENV (BUILT-IN)
─────────────────────────────────────────────────────────────
Create:     python3 -m venv venv
Activate:   source venv/bin/activate             (Mac/Linux)
            venv\Scripts\activate.bat            (Windows CMD)
            venv\Scripts\Activate.ps1            (PowerShell)
Deactivate: deactivate
Check:      which python3                         (Mac/Linux)
            where python                          (Windows)

─────────────────────────────────────────────────────────────
PIP
─────────────────────────────────────────────────────────────
Install:    pip install package
            pip install package==1.2.3
            pip install -r requirements.txt
Uninstall:  pip uninstall package
List:       pip list
Show:       pip show package
Freeze:     pip freeze > requirements.txt
Upgrade:    pip install --upgrade pip

─────────────────────────────────────────────────────────────
POETRY
─────────────────────────────────────────────────────────────
Install:    curl -sSL https://install.python-poetry.org | python3 -
New proj:   poetry new myproject
Init:       poetry init
Add:        poetry add package
            poetry add --group dev package
Remove:     poetry remove package
Install:    poetry install
Run:        poetry run python main.py
Shell:      poetry shell
Show:       poetry show --tree
Check:      poetry check
Update:     poetry update

─────────────────────────────────────────────────────────────
.ENV FILES
─────────────────────────────────────────────────────────────
Install:    pip install python-dotenv
Load:       from dotenv import load_dotenv; load_dotenv()
Access:     os.getenv("VAR_NAME", "default")
Never:      .env
Always:     .env.example

─────────────────────────────────────────────────────────────
GOLDEN RULES
─────────────────────────────────────────────────────────────
✅ One venv per project
✅ Always activate before working
✅ Never install packages globally
✅ Always add venv/ to .gitignore
✅ Always add .env to .gitignore
✅ Commit .env.example
✅ Use Poetry for new projects
✅ Commit poetry.lock (exact reproducibility)
✅ Use pip freeze > requirements.txt for pip projects
```

---
## Knowledge Check

1. Why do virtual environments exist?
2. What command creates a virtual environment?
3. How do you know if a virtual environment is active?
4. What does "source venv/bin/activate" actually do?
5. Why should you never use global Python for projects?
6. What is the difference between `pip install` and `pip install -r requirements.txt`?
7. What does `pip freeze` do?
8. What is the difference between `.env` and `.env.example`?
9. Why should `.env` never be committed to Git?
10. What is `pyproject.toml` and what does it replace?
11. What is `poetry.lock` and should you commit it?
12. What is the difference between `poetry add` and `poetry add --group dev`?
13. Why is `pip freeze > requirements.txt` imperfect?
14. What does `poetry install` do after `git clone`?

---

## Hands-On Exercise - Build It Right Now
```bash
# Complete exercise: Set up a real project from scratch

# 1. Create project structure
mkdir ~/projects/my_first_api
cd ~/projects/my_first_api

# 2. Initialize with Poetry
poetry init --name "my-first-api" \
    --description "My first backend API" \
    --author "Your Name <you@email.com>" \
    --python "^3.12" \
    --no-interaction

# 3. Add dependencies
poetry add fastapi "uvicorn[standard]" python-dotenv
poetry add --group dev pytest black ruff

# 4. Create project structure
mkdir -p src/myapi tests scripts
touch src/myapi/__init__.py
touch src/myapi/main.py
touch src/myapi/config.py
touch tests/__init__.py
touch tests/test_main.py
touch .env
touch .env.example
touch README.md

# 5. Write config.py
cat > src/myapi/config.py << 'EOF'
from dotenv import load_dotenv
import os

load_dotenv()

class Settings:
    APP_NAME: str = os.getenv("APP_NAME", "MyFirstAPI")
    DEBUG: bool = os.getenv("DEBUG", "True").lower() == "true"
    HOST: str = os.getenv("HOST", "0.0.0.0")
    PORT: int = int(os.getenv("PORT", "8000"))

settings = Settings()
EOF

# 6. Write main.py
cat > src/myapi/main.py << 'EOF'
from fastapi import FastAPI
from .config import settings

app = FastAPI(
    title=settings.APP_NAME,
    debug=settings.DEBUG
)

@app.get("/")
def root():
    return {
        "message": f"Welcome to {settings.APP_NAME}!",
        "debug": settings.DEBUG,
        "version": "1.0.0"
    }

@app.get("/health")
def health():
    return {"status": "healthy"}
EOF

# 7. Write .env
cat > .env << 'EOF'
APP_NAME=MyFirstAPI
DEBUG=True
HOST=0.0.0.0
PORT=8000
EOF

# 8. Write .env.example
cat > .env.example << 'EOF'
APP_NAME=MyFirstAPI
DEBUG=False
HOST=0.0.0.0
PORT=8000
EOF

# 9. Write .gitignore
echo ".venv/
venv/
.env
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
.ruff_cache/" > .gitignore

# 10. Install and run
poetry install
poetry run uvicorn src.myapi.main:app --reload

# Visit: http://localhost:8000
# Visit: http://localhost:8000/docs  (auto API docs!)
# Visit: http://localhost:8000/health

# 11. Initialize Git
git init
git add .
git commit -m "feat: initialize my first API project"
```

---