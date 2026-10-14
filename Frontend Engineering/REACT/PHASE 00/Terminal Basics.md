## Concept Explanation

Since you come from a Python backend background, you likely already have solid command-line experience. This section will focus on terminal usage specifically relevant to frontend workflows — file navigation, running Node-based tools, and conventions you will use constantly once we start building projects.

### Shell Types

- macOS: Uses zsh by default since Catalina (2019)
- Linux: Usually bash by default, though many developers switch to zsh

Check your current shell:

```bash
echo $SHELL
```

## Essential Navigation Commands

| Command | Purpose |
| --- | --- |
| `pwd` | Print current working directory |
| `ls` | List files/folders in current directory |
| `ls -la` | List all files including hidden ones, with details |
| `cd foldername` | Move into a folder |
| `cd ..` | Move up one directory |
| `cd ~` | Go to home directory |
| `cd -` | Go to previous directory |

## File and Folder Manipulation

| Command | Purpose |
| --- | --- |
| `mkdir foldername` | Create a new folder |
| `mkdir -p path/to/nested/folder` | Create nested folders at once |
| `touch filename.js` | Create a new empty file |
| `rm filename.js` | Delete a file |
| `rm -rf foldername` | Delete a folder and its contents (use with caution) |
| `cp source destination` | Copy a file |
| `cp -r source destination` | Copy a folder and its contents |
| `mv source destination` | Move or rename a file/folder |

## Viewing File Contents

| Command | Purpose |
| --- | --- |
| `cat filename` | Print entire file content to terminal |
| `head filename` | Show first 10 lines of a file |
| `tail filename` | Show last 10 lines of a file |
| `tail -f filename` | Continuously watch a file for new content (useful for logs) |

## Searching and Finding

| Command | Purpose |
| --- | --- |
| `find . -name "*.js"` | Find all JavaScript files in current directory and subfolders |
| `grep "searchterm" filename` | Search for text inside a file |
| `grep -r "searchterm" .` | Search for text recursively in all files in current directory |

## Process and System Management

| Command | Purpose |
| --- | --- |
| `ps aux` | List running processes |
| `kill PID` | Terminate a process by its ID |
| `lsof -i :3000` | Find what process is using port 3000 (common when a dev server won't start because the port is occupied) |
| `kill -9 $(lsof -t -i:3000)` | Force kill whatever is running on port 3000 |

## Frontend-Specific Terminal Workflow

In real frontend development, your terminal workflow typically looks like this:

```bash
cd my-project              # navigate into project
npm install                # install dependencies
npm run dev                # start development server
```

You will usually keep multiple terminal tabs/splits open simultaneously: one running the dev server, one for Git commands, and one free for running other commands (installing packages, checking logs, etc.).

## Terminal Shortcuts Worth Knowing

| Shortcut | Purpose |
| --- | --- |
| `Ctrl + C` | Stop currently running process (e.g., stop dev server) |
| `Ctrl + R` | Search command history |
| `Tab` | Auto-complete file/folder names and commands |
| `Up Arrow` | Cycle through previous commands |
| `clear` | Clear terminal screen |

## Aliases (Productivity Habit)

Professional developers customize their shell with aliases to save time. These go in `~/.zshrc` (macOS default) or `~/.bashrc` (Linux):

```bash
alias ll="ls -la"
alias gs="git status"
alias dev="npm run dev"
```

After adding aliases, reload the shell configuration:

```bash
source ~/.zshrc
```

## Basic Syntax Example

A typical sequence when starting work on a new frontend project:

```bash
cd ~/Projects
mkdir portfolio-site
cd portfolio-site
touch index.html style.css script.js
ls
```

Checking and freeing a port when a dev server fails to start because it is already in use:

```bash
lsof -i :5173
kill -9 <PID_shown_in_output>
```

## Industry-Level Code Snippet

In real projects, terminal usage often extends into shell scripts that automate repetitive setup tasks. Here is a realistic setup script a frontend team might include in a repository to onboard new developers quickly.

```bash
#!/bin/bash
# setup.sh
# Run once when cloning the repository for the first time
# Usage: chmod +x setup.sh && ./setup.sh

echo "Checking Node.js version..."
if ! command -v node &> /dev/null
then
    echo "Node.js not found. Please install nvm and Node.js before continuing."
    exit 1
fi

REQUIRED_NODE_VERSION=$(cat .nvmrc)
echo "Using Node version from .nvmrc: $REQUIRED_NODE_VERSION"
nvm install "$REQUIRED_NODE_VERSION"
nvm use "$REQUIRED_NODE_VERSION"

echo "Installing dependencies..."
npm install

echo "Checking for environment file..."
if [ ! -f .env ]; then
    cp .env.example .env
    echo ".env file created from .env.example. Please update values as needed."
fi

echo "Setup complete. Run 'npm run dev' to start the development server."
```

```bash
# Common package.json script that wraps multiple terminal commands into one,
# a pattern used in nearly every real frontend repository

"scripts": {
  "clean": "rm -rf dist node_modules",
  "fresh-install": "npm run clean && npm install",
  "dev": "vite",
  "find-port": "lsof -i :5173"
}
```

This kind of onboarding script is standard in professional repositories — new engineers run one command, and the environment is fully ready, rather than manually executing a dozen terminal commands individually.
