# Syncing `beta` and `main` after parallel changes

Sometimes work lands on **`main`** and **`beta`** separately (e.g. two agents or two PRs). GitHub will show **both** branches ahead of the other’s merge-base. You only want **one** combined history.

## Recommended policy

Treat **`beta`** as the integration branch: **merge `main` into `beta` first**, resolve conflicts once, then **merge `beta` into `main`** so production’s `main` matches what you tested on `beta`.

If instead **everything good lives on `main`** and `beta` is stale, you can **hard-reset `beta` to `main`** (see [Discard one branch entirely](#discard-one-branch-entirely)).

## Standard flow (merge, keep one resolution per conflict)

Run from a clone of **WebApp** (paths assume repo root).

```bash
git fetch origin

# See divergence
git log --oneline --left-right origin/main...origin/beta
```

### 1. Integrate on `beta`

```bash
git checkout beta
git pull origin beta
git merge origin/main -m "Merge main into beta to reconcile parallel work"
```

- If Git reports **conflicts**: open each file, pick **one** outcome (do not commit both competing features unless you intend to merge them by hand).
- Useful commands:
  - `git status` — list conflicted files
  - For a whole file, take **beta’s** version: `git checkout --ours -- path/to/file` (when merging **main** into **beta**, *ours* is `beta`)
  - Take **main’s** version: `git checkout --theirs -- path/to/file`
  - After editing: `git add path/to/file`
- Finish: `git commit` (if merge opened an editor, save the merge message)

```bash
git push origin beta
```

### 2. Ship to `main`

```bash
git checkout main
git pull origin main
git merge origin/beta -m "Merge beta into main"
git push origin main
```

If this is a **fast-forward**, `main` simply moves to `beta`’s tip—no merge commit.

### 3. Server checkouts

```bash
cd /path/to/WebApp/beta   && git pull origin beta
cd /path/to/WebApp/production && git checkout main && git pull origin main
```

## Discard one branch entirely

**Only if** one branch’s commits should be thrown away and the other branch is correct:

- Make **`beta`** match **`main`** (destructive for unique commits on `beta`):

  ```bash
  git checkout beta
  git fetch origin
  git reset --hard origin/main
  git push origin beta --force-with-lease
  ```

- Or make **`main`** match **`beta`** (destructive for unique commits on `main`—rare and risky if production already deployed):

  ```bash
  git checkout main
  git fetch origin
  git reset --hard origin/beta
  git push origin main --force-with-lease
  ```

Use **`--force-with-lease`** so you do not overwrite new commits someone else pushed.

## Submodule `docs/`

If only **`docs/`** changed on one branch, bump the submodule on **`beta`** first, then merge **`beta` → `main`** as usual. See [publishing-docs.md](publishing-docs.md).
