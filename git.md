
# Git & GitHub CLI

Distributed version control + `gh` for GitHub operations from the terminal.

- [Setup / config](#setup--config)
- [Core workflow](#core-workflow)
- [Conventional commits](#conventional-commits)
- [Branches](#branches)
- [Remotes](#remotes)
- [Inspect history](#inspect-history)
- [Undo & fix](#undo--fix)
- [Stash](#stash)
- [Rebase & merge](#rebase--merge)
- [gh: repos](#gh-repos)
- [gh: branches & PRs](#gh-branches--prs)
- [Clone](#clone)
- [Typical workflows](#typical-workflows)
- [Gotchas](#gotchas)

## Setup / config
```bash
git config --global user.name "Ahmad"
git config --global user.email "ahmadmmsa@gmail.com"
git config --global init.defaultBranch main      # future `git init` uses main, not master
git config --global pull.rebase false            # merge on pull (true = rebase instead)
git config --global core.editor "nano"
git config --list                                # show all effective config
gh auth login                                    # authenticate the gh CLI (browser or token)
gh auth status                                   # confirm who you're logged in as
```

## Core workflow
```bash
git init                            # initialize a new local repo
git status                          # what's changed / staged / untracked
git status -s                       # short format (M/A/?? columns)
git add .                           # stage all changes
git add <file>                      # stage a specific file only
git add -p                          # stage hunk-by-hunk (interactive)
git commit -m "msg"                 # commit staged changes
git commit -am "msg"               # stage tracked files + commit in one step (skips untracked)
git push                            # send commits to the linked remote/branch
git push -u origin main             # first push: set upstream so later `git push` needs no args
git pull                            # fetch + merge remote changes
git log                             # full commit history
```

## Conventional commits
`type(scope): summary` — keep the summary imperative and under ~50 chars.
```bash
# feat: implement global error handler
# feat(db): add user schema
# feat(db): integrate mongoose connection
# feat(ui): update button color

# fix(db): prevent duplicate key errors
# fix: handle async errors in express routes
# fix: sanitize user input to prevent injection

# perf: optimize database query for user lookup

# test: add unit tests for auth service
# test(api): cover edge cases for user routes

# chore: update dependencies
# chore(deps): bump axios from 1.6 to 1.7
# chore: configure eslint and prettier
# chore: update gitignore

# build: configure TypeScript compiler
# build: add webpack bundling

# ci: add GitHub Actions workflow for Docker build

# docs: update API usage in README
# docs: add environment setup instructions

# style: apply prettier formatting   (formatting only, no logic change)
# refactor: extract validation into helper  (no behavior change)

# Examples
git commit -m "docs: update README setup"
git commit -m "fix: login error on empty password" \
           -m "Add validation to prevent null value from being sent to API"   # 2nd -m = body
```
> Common types: `feat` `fix` `perf` `refactor` `test` `chore` `build` `ci` `docs` `style`. `feat`/`fix` are what users see in changelogs; the rest are housekeeping.

## Branches
```bash
git branch                          # list local branches (* = current)
git branch -vv                      # list + show upstream tracking + last commit
git branch -a                       # include remote-tracking branches
git checkout -b <branch>            # create + switch to new branch
git switch <branch>                 # switch (modern, clearer than checkout)
git switch -c <branch>              # create + switch (modern form of checkout -b)

git branch -m <old> <new>           # rename a branch
git branch -d <branch>              # delete local branch (safe: refuses if unmerged)
git branch -D <branch>             # force-delete local branch (even if unmerged)

# rename master -> main and re-point upstream
git branch -m master main
git branch --set-upstream-to=origin/main main
git branch -vv                      # verify the new tracking
```

## Remotes
```bash
git remote -v                       # list remotes + URLs
git remote add origin <url>         # link local repo to a remote (e.g. GitHub)
git remote set-url origin <url>     # change the remote URL (e.g. https -> ssh)
git remote remove origin
git fetch origin                    # download remote refs WITHOUT merging
git push --force-with-lease         # safer force-push: aborts if remote moved under you
git push --force                    # force-push (overwrites remote history — dangerous)
```
> Prefer `--force-with-lease` over `--force`: it refuses to clobber commits you haven't seen, so you can't silently nuke a teammate's push.

## Inspect history
```bash
git log --oneline --graph --all     # compact visual history of all branches
git log -p <file>                   # history with full diffs for a file
git log --stat                      # files changed + insertion/deletion counts
git log --author="Ahmad" --since="2 weeks ago"
git show <commit>                   # full detail + diff of one commit
git diff                            # unstaged changes vs working tree
git diff --staged                   # staged changes vs last commit
git diff <branchA>..<branchB>       # difference between two branches
git blame <file>                    # who last touched each line
```

## Undo & fix
```bash
git commit --amend -m "new msg"     # rewrite the LAST commit (message or add staged files)
git restore <file>                  # discard unstaged changes to a file
git restore --staged <file>         # unstage (keep the edits)
git reset --soft HEAD~1             # undo last commit, KEEP changes staged
git reset --mixed HEAD~1            # undo last commit, keep changes unstaged (default)
git reset --hard HEAD~1             # undo last commit, DISCARD changes (destructive)
git revert <commit>                 # make a NEW commit that undoes <commit> (safe for shared history)
git checkout -- <file>              # (legacy) discard local changes to file
git reflog                          # log of where HEAD has been — your undo safety net
```
> On a **shared** branch use `git revert` (adds a commit), never `reset --hard` + force-push. `git reflog` can recover commits you thought you destroyed.

## Stash
```bash
git stash                           # shelve dirty changes, return to clean tree
git stash -u                        # include untracked files
git stash list
git stash pop                       # re-apply most recent stash + drop it
git stash apply stash@{1}           # re-apply a specific stash, keep it in the list
git stash drop stash@{0}
git stash clear                     # delete all stashes
```

## Rebase & merge
```bash
git merge <branch>                  # merge <branch> into current (creates a merge commit)
git rebase main                     # replay current branch's commits on top of main (linear history)
git rebase -i HEAD~3                # interactively squash/reword/reorder last 3 commits
git rebase --continue               # after resolving conflicts
git rebase --abort                  # bail out, back to pre-rebase state
git cherry-pick <commit>            # apply a single commit from elsewhere onto current branch
```
> Golden rule: **don't rebase commits that are already pushed/shared** — it rewrites hashes and breaks everyone who pulled them. Rebase local work, merge shared work.

## gh: repos
```bash
gh repo create                                              # interactive
gh repo create my-repo --public --source=. --remote=origin --push   # create from current dir + push
gh repo create my-repo --public --source=. --push --default-branch main
gh repo list                                               # your repos
gh repo clone <name>                                       # clone to current dir
gh browse                                                  # open current repo in browser
gh repo fork                                               # fork someone else's repo to your account
gh repo delete <name>                                      # delete (prompts to confirm)
gh repo delete <owner>/<name> --yes                        # delete without prompt
```
> `gh repo delete` is **irreversible** and needs the `delete_repo` scope (`gh auth refresh -s delete_repo`).

## gh: branches & PRs
```bash
# list branches via the API
gh api repos/:owner/:repo/branches --jq '.[].name'         # :owner/:repo auto-filled in a repo
gh api repos/<owner>/<repo>/branches --jq '.[].name'

# pull requests
gh pr create --fill                  # open a PR, auto-fill title/body from commits
gh pr create --base main --head feature-x --title "..." --body "..."
gh pr list                           # open PRs
gh pr status                         # PRs relevant to you
gh pr view <num> --web               # open a PR in the browser
gh pr checkout <num>                 # check out a PR's branch locally
gh pr merge --delete-branch          # merge current PR + clean up the branch
gh pr merge <num> --squash --delete-branch
```
> Deleting a remote branch is `git push origin --delete <branch>` (or `gh pr merge --delete-branch`), not a `gh repo delete` flag — that command deletes the whole repository.

## Clone
```bash
gh repo clone <owner>/my-app && cd my-app          # via gh (uses your auth)
git clone https://github.com/<owner>/my-app.git    # via https
git clone git@github.com:<owner>/my-app.git        # via ssh (needs an ssh key on GitHub)
git clone --depth 1 <url>                          # shallow: latest commit only (fast, big repos)
```

## Typical workflows
```bash
# Feature branch -> PR -> merge
git switch -c feature-x
git add .
git commit -m "feat: add feature x"
git push -u origin feature-x
gh pr create --fill
gh pr merge --delete-branch

# Quick fix on an existing repo
git add .
git commit -m "fix: correct css spacing"
git push

# Brand-new project up to GitHub
git init
git add .
git commit -m "chore: initial commit"
gh repo create my-app --public --source=. --remote=origin --push
```

## Gotchas
- **`git push` says "no upstream branch"** — first push needs `git push -u origin <branch>` to set tracking; after that bare `git push` works.
- **`--force` overwrote a teammate's work** — use `--force-with-lease` instead; it aborts if the remote moved since your last fetch.
- **`reset --hard` ate your changes** — check `git reflog`; the commit is usually still recoverable for ~90 days.
- **Rebased a shared branch** — everyone else now has divergent history. Rebase only local/unpushed commits.
- **`gh repo delete` permission denied** — refresh the scope: `gh auth refresh -s delete_repo`.
- **Committed a secret/.env** — removing it in a new commit isn't enough; it's still in history. Rotate the secret and rewrite history (`git filter-repo`) — don't just `rm` it.
- **`commit -am` skipped a new file** — `-a` only stages *tracked* files; new/untracked files still need `git add` first.
- **Wrong author on commits** — set `user.email` to your GitHub email or commits won't link to your profile / count in contributions.
