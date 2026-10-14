## Concept Explanation

`Node.js` is a JavaScript runtime that allows JavaScript to run outside the browser. For frontend engineers, Node.js itself is not usually used to build production websites directly, but it is absolutely essential because:

- npm (Node Package Manager) comes bundled with Node.js and is used to install every frontend library and tool (React, Tailwind, testing libraries, build tools, etc.)
- Build tools like Vite, Next.js, and TypeScript all run on Node.js
- Many development workflows (linting, formatting, bundling) are powered by Node.js scripts

## Why Version Management Matters

In professional environments, different projects often require different Node.js versions. A company's legacy project might require Node 16, while a new project requires Node 20. Installing Node.js directly (via the official installer) locks you into one global version, which causes conflicts.

The industry-standard solution is nvm (Node Version Manager), which allows you to install multiple Node.js versions and switch between them per project instantly.

## Installing nvm (macOS and Linux)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

After installation, restart your terminal or run:

```bash
source ~/.zshrc    # if using zsh (default on macOS)
source ~/.bashrc   # if using bash (common on Linux)
```

Verify installation:

```bash
nvm --version
```

## Installing Node.js via nvm

```bash
nvm install --lts
```

This installs the latest LTS (Long Term Support) version, which is what professional teams use in production (not the "Current/Latest" cutting-edge version, which is less stable).

Set it as default:

```bash
nvm use --lts
nvm alias default lts/*
```

Verify:

```bash
node --version
npm --version
```

## Switching Node Versions Per Project

```bash
nvm install 18        # install a specific version
nvm use 18             # switch to it for current terminal session
nvm list               # see all installed versions
```

Many professional projects include a `.nvmrc` file in the repository root specifying the exact Node version required. When any developer runs `nvm use` inside that project folder, it automatically reads this file and switches to the correct version.

```text
18.20.4
(contents of a .nvmrc file)
```

## Understanding npm

npm is both a package manager and a command-line tool. Its core responsibilities:

- Installing third-party libraries (`npm install react`)
- Managing project dependencies via `package.json`
- Running project scripts (`npm run dev`, `npm run build`)
- Locking exact dependency versions via `package-lock.json`

### package.json

This file is the identity card of any JavaScript/frontend project. It lists:

- Project metadata (name, version)
- Dependencies (libraries needed to run the app)
- devDependencies (tools needed only during development, like ESLint)
- Scripts (custom commands like `start`, `build`, `test`)

## Basic Syntax Example

Initializing a new project and installing a package:

```bash
mkdir my-first-project
cd my-first-project
npm init -y          # creates package.json with default values
npm install lodash   # installs a library and adds it to package.json
```

Example of a basic package.json after installing one package:

```json
{
  "name": "my-first-project",
  "version": "1.0.0",
  "dependencies": {
    "lodash": "^4.17.21"
  }
}
```

Running a script defined in package.json:

```bash
npm run dev
```

## Industry-Level Code Snippet

A realistic package.json from a production-grade React/TypeScript project, showing proper separation of dependencies, devDependencies, and useful scripts.

```json
{
  "name": "enterprise-dashboard",
  "version": "2.4.1",
  "private": true,
  "engines": {
    "node": ">=18.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .ts,.tsx --report-unused-disable-directives --max-warnings 0",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\"",
    "test": "vitest run",
    "test:watch": "vitest",
    "prepare": "husky install"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.26.0",
    "@tanstack/react-query": "^5.51.1",
    "zustand": "^4.5.4",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "typescript": "^5.5.3",
    "vite": "^5.3.4",
    "eslint": "^9.8.0",
    "prettier": "^3.3.3",
    "vitest": "^2.0.4",
    "husky": "^9.1.4",
    "lint-staged": "^15.2.7"
  },
  "lint-staged": {
    "src/**/*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

Key professional details visible in this file:

- `"engines"` enforces a minimum Node.js version for anyone running the project
- Dependencies vs devDependencies are clearly separated (runtime necessities vs development-only tools)
- `"^"` before version numbers allows automatic minor/patch updates but locks the major version, preventing breaking changes
- lint-staged ensures code is automatically linted and formatted before every commit (paired with Husky, covered in the Git phase)

```bash
# .nvmrc file content - ensures entire team uses the same Node version
18.20.4
```
