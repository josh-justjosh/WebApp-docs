# Publishing changes to this documentation (WebApp-docs submodule)

This directory is the **[WebApp-docs](https://github.com/josh-justjosh/WebApp-docs)** repository, embedded in the main WebApp repo as the **`docs/`** git submodule. Markdown here is **not** deployed by Laravel; it is versioned in GitHub and checked out on the server next to `beta/` and `production/` for humans (and agents) to read.

## When you change any file under `docs/`

1. **Commit and push in this repository (WebApp-docs)**  
   Work from a clone of WebApp where `docs/` is the submodule checkout, **or** a standalone clone of WebApp-docs.

   ```bash
   cd docs    # from WebApp root, or cd /path/to/WebApp-docs
   git status
   git add <files>
   git commit -m "docs: concise summary of the change"
   git push origin main
   ```

   Use **SSH** remotes on the server if HTTPS credential helpers are unavailable (`git remote set-url origin git@github.com:josh-justjosh/WebApp-docs.git`).

2. **Point the WebApp repo at the new submodule commit**  
   The parent app (`WebApp`) records which commit of WebApp-docs it uses. Bump that pointer on **`beta`** first, then ship to production via **`main`**.

   ```bash
   cd /path/to/WebApp/beta   # or production while testing; usually beta
   git checkout beta
   git add docs
   git commit -m "chore(docs): bump WebApp-docs submodule"
   git push origin beta
   ```

3. **Ship the submodule bump to production (when ready)**  
   Merge **`beta` → `main`** in the WebApp repo (PR or local merge + `git push origin main`). On the server, update the production checkout:

   ```bash
   cd /path/to/WebApp/production
   git checkout main
   git pull origin main
   ```

   That updates **`docs/`** to the new submodule SHA along with the rest of **`main`**. Do the same in **`beta/`** if that clone should match (`git checkout beta && git pull origin beta`).

## Checklist

| Step | Where | Action |
|------|--------|--------|
| 1 | `docs/` (this repo) | `git commit` + `git push origin main` |
| 2 | WebApp `beta/` | `git add docs` + commit + `git push origin beta` |
| 3 | WebApp | Merge `beta` → `main` when ready |
| 4 | Server `production/` | `git pull` on `main` (and `beta/` if you want both checkouts aligned) |

## Notes

- **Relative links** in files such as [AgentContext.md](AgentContext.md) that use `../` assume this tree lives at **`WebApp/docs/`** inside the WebApp working copy. They target app files (e.g. `../docker/Dockerfile`). Browsing **only** WebApp-docs on GitHub will not resolve those paths to the Laravel repo.
- **AgentContext.md** is aimed at AI agents and operators; procedural deploy steps stay in [deployment.md](deployment.md) and [deploy-status.md](deploy-status.md).
