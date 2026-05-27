```bash



git commit -m
#style: apply prettier formatting

# feat: implement global error handler
# feat(db): add user schema
# feat(db): integrate mongoose connection
# feat(ui): update button color

# fix(db): prevent duplicate key errors
# fix: handle async errors in express routes
# fix: sanitize user input to prevent injection
# fix: handle async errors in express routes

# perf: optimize database query for user lookup

# test: add unit tests for auth service
# test(api): cover edge cases for user routes

# chore: update dependencies
# chore: upgrade express to latest version
# chore(deps): bump axios from 1.6 to 1.7
# chore: add .env.example file
# chore: configure eslint and prettier
# chore: update gitignore
# chore: add prettier config
# chore: run prettier

# build: configure TypeScript compiler
# build: add webpack bundling

# ci: add GitHub Actions workflow for Docker build

# docs: update API usage in README
# docs: add environment setup instructions

# Examples
git commit -m "docs: update README setup"
git commit -m "fix: login error on empty password" \
#           -m "Add validation to prevent null value from being sent to API"
```

git
```bash
git init
#Initializes a new local Git repo.
git status
#Shows which files are changed/unstaged.
git add .
#Stages all changes for the next commit.
git add <file>
#Stages a specific file only.
git commit -m "msg"
#Saves your staged changes with a note.
git log
#Shows a history of all your commits.
git remote add origin <url>
#Links your local repo to GitHub.
git push -u origin main
git push --force
#Sends your local commits to GitHub.
git pull
#Fetches & merges changes from GitHub.
git config --global init.defaultBranch main
#future git init will use main
```


gh
```bash
gh repo create
gh repo create my-repo --public --source=. --remote=origin --push
gh repo create my-repo --public --source=. --push --default-branch main
gh repo delete <name>
# Deletes a repository from GitHub (requires confirmation).
gh repo list
# Shows all Github repositories.
gh repo clone <name>
# Downloads a repository to pc..
gh browse
# opens the current GitHub project's page in browser.
gh repo fork
# Creates a copy of someone else's repo in your account.
```


Branch Management 
```bash
# List Branches
gh api repos/:owner/:repo/branches --jq '.[].name'
gh api repos/gitusername/my-app/branches --jq ".[].name"
# Delete Branch
gh repo delete <repo> --branch <branch-name> --yes
# Delete Local Branch
git branch -d <branch-name>
```


Examples:
```bash
gh repo delete my-app
gh repo delete gitusername/odoo --yes
git init 
git add . 
git commit -m "more css updates"
git push
```


change branch name
```bash
git branch -m master main
git branch --set-upstream-to=origin/main main
git status
#to verify
git branch -vv
```

Typical Workflow (Minimal)
```bash
git checkout -b feature-x
git add .
git commit -m "feat: add feature x"
git push -u origin feature-x
gh pr create --fill
gh pr merge --delete-branch
```

Clone the repository
```bash
gh repo clone gitusername/my-app
cd my-app
Or with Git:
git clone https://github.com/gitusername/my-app.git
cd my-app
```