# Docs on GitHub Pages

Added a workflow that builds the mdBook in `rust-varlap/docs/` and publishes it to GitHub Pages, 2026-11-08, on branch `add-readthedocs-yaml`.

**Status:** the changes are in the working tree and are **not committed**. The workflow has **not been run** on GitHub yet. `mdbook build docs` succeeds locally with the new `book.toml`.

The site will be at **https://jchchiu.github.io/rust-varlap/**.

## Files changed

| File | Change |
|---|---|
| `.github/workflows/docs.yml` | New. Builds the book and deploys it to Pages |
| `docs/book.toml` | Added `site-url`, `git-repository-url` and `edit-url-template` under `[output.html]` |

## The workflow (`docs.yml`)

**Triggers:**
- A push to `main` that touches `docs/**` or the workflow file itself. Code-only pushes don't rebuild the site.
- `workflow_dispatch`, to run it by hand from any branch.

**Jobs:**

| Job | What it does |
|---|---|
| `build` | Checks out the repo and installs `mdbook` 0.5.4 and `mdbook-tabs` 1.0.5. It uses `taiki-e/install-action`, which downloads pre-built binaries instead of compiling them. It then runs `mdbook build docs` and uploads `docs/book` as the Pages artifact. |
| `deploy` | Publishes the artifact with `actions/deploy-pages`. Permissions are `pages: write` and `id-token: write`, in the `github-pages` environment. |

**Notes:**
- **Pinned versions:** the mdbook and mdbook-tabs versions match the ones installed locally (`~/.cargo/bin`), so the site renders the same as a local build. To upgrade, change `MDBOOK_VERSION` and `MDBOOK_TABS_VERSION` at the top of the workflow.
- **Concurrency:** set to group `pages` with `cancel-in-progress: false`. Only one deploy runs at a time, and a deploy that's already running is never cut off.
- **No `gh-pages` branch:** the workflow uses the Pages artifact flow, so the built HTML never gets committed anywhere.
- **Tabs preprocessor:** `mdbook-tabs` must be installed for the build to work, because `book.toml` has `[preprocessor.tabs]`. The tab CSS and JS are already committed in `docs/theme/`, so CI doesn't need to run `mdbook-tabs install`.

## `book.toml` additions

```toml
site-url = "/rust-varlap/"
git-repository-url = "https://github.com/jchchiu/rust-varlap"
edit-url-template = "https://github.com/jchchiu/rust-varlap/edit/main/docs/{path}"
```

- **`site-url`:** Pages serves the site under `/rust-varlap/`, not `/`. Without this setting, the links on `404.html` would point to the domain root and break. I checked that the built `404.html` now links to `/rust-varlap/`.
- **`git-repository-url`:** adds a GitHub icon to the top bar.
- **`edit-url-template`:** adds an "edit this page" icon. The links resolve to the real files, for example `edit/main/docs/src/0_introduction/README.md`.

## Getting the website up and running

1. **Commit and push** `.github/workflows/docs.yml` and `docs/book.toml`.
2. **Turn on Pages in the repo settings.** This step is required, and it has to happen before the first deploy or the deploy job fails.
   - Go to https://github.com/jchchiu/rust-varlap/settings/pages.
   - Under **Build and deployment → Source**, choose **GitHub Actions**, not "Deploy from a branch".
3. **Run the workflow for the first time.** Either:
   - Merge the branch into `main` (the push runs the workflow, because `docs/**` changed), or
   - Run it now: **Actions → Deploy docs → Run workflow**, and pick the branch. You can also run `gh workflow run docs.yml --ref add-readthedocs-yaml`, but only once the workflow file exists on the default branch. Until then, GitHub only lists workflows that are on `main`.
4. **If the deploy job is rejected**, with a message like "Branch … is not allowed to deploy to github-pages": go to **Settings → Environments → github-pages → Deployment branches and tags**. By default it only allows `main`, so either run from `main` or add your branch there.
5. **Check the site:** open https://jchchiu.github.io/rust-varlap/. The deploy job's summary also links to it. The first deploy can take a minute or two to appear.
6. **Optional:** set the repo's About → Website field to the Pages URL, and link to the docs from `README.md`.

After that, every push to `main` that changes `docs/` redeploys the site automatically.

### Previewing locally

```bash
cd rust-varlap
mdbook serve docs --open   # live-reloads at http://localhost:3000
```

Locally the site is served at `/`, not `/rust-varlap/`. Normal pages work fine. Only the 404 page's links point at `/rust-varlap/`.

## Loose ends (not changed)

- **`.readthedocs.yaml`:** it still configures Sphinx (`docs/conf.py`, which doesn't exist). If GitHub Pages replaces Read the Docs, delete it. If you want both, change it to build mdBook:
  - Use `build.commands` to install mdbook and mdbook-tabs and copy `docs/book` to `$READTHEDOCS_OUTPUT/html`.
  - Change `site-url` too, because Read the Docs serves under `/en/latest/`.
- **Branch name:** the branch is called `add-readthedocs-yaml`, which no longer matches what it does.
- **Docs links:** `installation.md` and the other docs pages could link back to the hosted site, for example from `README.md`.
- See also `RELEASE_WORKFLOW.md` for the release workflow changes on the same branch, which are also uncommitted.
