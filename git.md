

change branch from master to main

```bash
git branch -m master main
```

> -m move

```bash
git push -u origin main
```

> -u sets the upstream tracking,

Go to GitHub:

- GitHub: Settings > Branches change the Default branch to main.
- GitLab: Settings > Repository > Default branch.

delete
```bash
git push origin --delete master
```