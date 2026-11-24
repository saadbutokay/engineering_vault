## Setup
```bash
brew install git

git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false          # or true, to rebase on pull
git config --global core.editor "code --wait"  # VS Code
git config --global credential.helper osxkeychain
git config --list --show-origin
```

**Global ignore for Mac junk**
```bash
echo ".DS_Store" >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
```

---
## Start a `repo`
```bash
git init
git clone <url>
git clone <url> myfolder
git clone --depth 1 <url>        # shallow clone (latest commit only)
```

---
## Daily workflow
```bash
git status
git status -s                    # short format
git add file.py
git add .                        # everything in the current dir
git add -p                       # stage interactively, hunk by hunk
git commit -m "Message"
git commit -am "Message"         # add tracked files + commit
git commit --amend               # edit last commit (message and/or staged changes)
git commit --amend --no-edit
```

---
## Viewing changes and history
```bash
git diff                         # unstaged changes
git diff --staged                # staged changes
git diff main..feature           # between branches
git log
git log --oneline --graph --decorate --all
git log -p file.py               # history with diffs for a file
git log --author="Name"
git log -S"search_term"          # commits that added/removed a string
git show <commit>
git blame file.py
git reflog                       # history of HEAD (lifesaver)
```

---
## Branching
```bash
git branch                       # list local
git branch -a                    # list all, including remote
git branch feature               # create
git switch feature               # switch
git switch -c feature            # create + switch
git branch -m new-name           # rename current
git branch -d feature            # delete (safe)
git branch -D feature            # force delete
git branch -vv                   # show tracking info
```

---
## Merging and rebasing
```bash
git merge feature
git merge --no-ff feature        # always create a merge commit
git merge --abort

git rebase main
git rebase -i HEAD~4             # interactive: squash, reword, drop, reorder
git rebase --continue
git rebase --abort

git cherry-pick <commit>
```
**Conflicts:** edit the files, remove the `<<<<<<<` markers, then `git add <file>` followed by `git commit` (merge) or `git rebase --continue`.

---
## Remotes
```bash
git remote -v
git remote add origin <url>
git remote set-url origin <new-url>

git fetch
git fetch --prune                # also remove stale remote branches
git pull
git pull --rebase
git push
git push -u origin feature       # push and set upstream
git push --force-with-lease      # safer force push
git push origin --delete feature # delete remote branch
```

---
## Undoing things

|Goal|Command|
|---|---|
|Discard unstaged changes in a file|`git restore file.py`|
|Unstage a file|`git restore --staged file.py`|
|Undo last commit, keep changes staged|`git reset --soft HEAD~1`|
|Undo last commit, keep changes unstaged|`git reset HEAD~1`|
|Undo last commit, **discard changes**|`git reset --hard HEAD~1`|
|Safely undo a pushed commit|`git revert <commit>`|
|Reset to match remote|`git fetch && git reset --hard origin/main`|
|Remove untracked files (preview)|`git clean -n`|
|Remove untracked files and dirs|`git clean -fd`|
|Recover a "lost" commit|`git reflog`, then `git reset --hard <hash>`|

⚠️ `reset --hard` and `clean -fd` permanently discard uncommitted work.

---
## Stash
```bash
git stash
git stash -u                     # include untracked files
git stash push -m "message"
git stash list
git stash pop                    # apply + remove
git stash apply                  # apply + keep
git stash drop stash@{0}
git stash clear
```

---
## Tags
```bash
git tag v1.0.0
git tag -a v1.0.0 -m "Release 1.0.0"
git tag
git push origin v1.0.0
git push --tags
git tag -d v1.0.0
```

---
## Files
```bash
git rm file.py                   # delete + stage
git rm --cached file.py          # untrack but keep on disk
git mv old.py new.py
git ls-files
```

**Stop tracking a file that's now in `.gitignore`:**
```bash
git rm --cached file
git commit -m "Stop tracking file"
```

---
## Worktrees and bisect
```bash
git worktree add ../hotfix hotfix-branch   # second working dir, same repo
git worktree list
git worktree remove ../hotfix

git bisect start
git bisect bad                   # current commit is broken
git bisect good <commit>         # known good commit
# test, then mark good/bad until found
git bisect reset
```

---
## Useful aliases
```bash
git config --global alias.st "status -s"
git config --global alias.co switch
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 --stat"
git config --global alias.undo "reset --soft HEAD~1"
```

---
## SSH key for GitHub (Mac)
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
pbcopy < ~/.ssh/id_ed25519.pub   # copy key, then paste into GitHub settings
ssh -T git@github.com            # test
```

Or use the GitHub CLI:
```bash
brew install gh
gh auth login
```

---
## Common situations
**Committed to the wrong branch**
```bash
git switch correct-branch
git cherry-pick <commit>
git switch wrong-branch
git reset --hard HEAD~1
```

**Edit the last commit message**
```bash
git commit --amend -m "New message"
```

**Squash the last 3 commits**
```bash
git rebase -i HEAD~3    # mark later commits as "squash" or "s"
```

**Pull request is behind main**
```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
```
**Committed a secret:** rotate the secret first. Removing it from history needs `git filter-repo` or BFG, and it's still exposed once pushed.

---
## Quick reference: `reset` modes

|Mode|HEAD|Staging|Working dir|
|---|---|---|---|
|`--soft`|moves|kept|kept|
|`--mixed` (default)|moves|reset|kept|
|`--hard`|moves|reset|**reset**|

---