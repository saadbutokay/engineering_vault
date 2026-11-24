## 1. `venv` (built-in)
```bash
# Create
python3 -m venv .venv

# Activate (zsh/bash)
source .venv/bin/activate

# Deactivate
deactivate

# Install packages
pip install requests
pip install -r requirements.txt

# Save dependencies
pip freeze > requirements.txt

# Upgrade pip
python -m pip install --upgrade pip

# Delete env
rm -rf .venv
```

**Tips:**
- Use `python3` on Mac. Plain `python` may not exist until the venv is active.
- Specific version: `python3.12 -m venv .venv` (requires that version to be installed).
- Add `.venv/` to `.gitignore`.

---
## 2. `uv` (fast, all-in-one)
```Bash
brew install uv
# or
curl -LsSf https://astral.sh/uv/install.sh | sh

uv self update          # update (curl install)
brew upgrade uv         # update (brew install)
```

### Project workflow (recommended)
```bash
uv init myproject               # new project
cd myproject
uv add requests                 # add dependency (creates .venv + uv.lock)
uv add --dev pytest ruff        # dev dependency
uv remove requests              # remove
uv sync                         # install from lockfile
uv lock --upgrade               # upgrade all locked versions
uv lock --upgrade-package requests
uv run python main.py           # run in the project env (no activation needed)
uv run pytest
uv tree                         # dependency tree
```

### Python versions
```bash
uv python list                  # available/installed
uv python install 3.12          # install a Python
uv python pin 3.12              # write .python-version
uv venv --python 3.12           # create venv with that version
```

### `venv`-style / `pip`-compatible workflow
```bash
uv venv                         # creates .venv
source .venv/bin/activate
uv pip install requests
uv pip install -r requirements.txt
uv pip freeze
uv pip compile requirements.in -o requirements.txt
uv pip sync requirements.txt
```

### Scripts and tools
```bash
uv run script.py                        # run a script
uv add --script script.py requests      # inline script dependencies
uvx ruff check .                        # run a tool without installing (like pipx run)
uv tool install ruff                    # install a CLI tool globally
uv tool list
uv tool upgrade --all
```

### Build/publish
```bash
uv build
uv publish
```

---
## 3. Poetry
```bash
brew install poetry
# or
curl -sSL https://install.python-poetry.org | python3 -

poetry self update
```

### Project workflow
```bash
poetry new myproject            # new project with layout
poetry init                     # add to an existing folder (interactive)
poetry add requests             # add dependency
poetry add --group dev pytest   # dev dependency
poetry remove requests
poetry install                  # install from lockfile
poetry install --no-root        # deps only, not the project itself
poetry update                   # update deps within constraints
poetry lock                     # re-lock
poetry show --tree              # dependency tree
poetry run python main.py       # run in the env
poetry run pytest
poetry shell                    # activate env (Poetry 2.x: needs shell plugin)
```

### Environment management
```bash
poetry env list
poetry env info
poetry env info --path
poetry env use 3.12             # pick a Python version
poetry env remove --all

# Put .venv inside the project (recommended)
poetry config virtualenvs.in-project true
```

### Export and publish
```bash
poetry build
poetry publish
poetry export -f requirements.txt -o requirements.txt   # needs poetry-plugin-export
```

---
## Common Mac gotchas
- **`python: command not found`**: use `python3`, or activate a venv first.
- **"externally-managed-environment" error**: Homebrew Python blocks global `pip install`. Use a venv, `uv`, or `pipx`/`uv tool`.
- **Apple Silicon wheels**: if a package fails to build, try `brew install <dep>` (e.g. `openssl`, `libffi`) or update to a newer package version.
- **Missing compiler**: `xcode-select --install`
- **Check which Python is active**: `which python` and `python --version`
- **Multiple Pythons**: use `uv python install`, or `brew install python@3.12`, or `pyenv`.

---
## Quick recommendations
- **New project?** Use **uv**.
- **Quick script or no installs allowed?** Use **venv**.
- **Existing team using Poetry?** Stay on **Poetry**.

---
## Migrating to `uv`
```bash
# From requirements.txt
uv init
uv add -r requirements.txt

# From Poetry (pyproject.toml uses [tool.poetry])
# Convert to [project] format manually or with a tool such as `migrate-to-uv`
```

---