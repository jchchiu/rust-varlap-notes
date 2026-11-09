# Publishing to Docker, crates.io and Bioconda

Written 2026-11-08. These instructions go with the release workflow changes in `RELEASE_WORKFLOW.md`, which are not committed yet.

| Target | How it gets published | Setup needed (once) |
|---|---|---|
| Docker (ghcr.io) | `publish-docker` job in `release.yml`, on every tag | Make the package public after the first push |
| crates.io | `publish-crate` job in `release.yml`, on every tag | Add the `CARGO_REGISTRY_TOKEN` repo secret |
| Bioconda | Manual pull request to `bioconda/bioconda-recipes`. Their bot bumps the version after that. | Write and submit the recipe |

Checked on 2026-11-08:
- **Crate name:** `rust-varlap` is not taken on crates.io (`cargo search` and `cargo info` found nothing).
- **Dry run:** `cargo publish --dry-run --locked` passes: 20 files, 38 KB compressed. The `exclude` in `Cargo.toml` keeps `test_data/` out of the crate.

---

## Docker (ghcr.io)

**Setup (once):**
1. Nothing to configure before the first push. The `publish-docker` job logs in with the built-in `GITHUB_TOKEN` and has `packages: write` permission.
2. Push a tag (see [Releasing](#releasing-docker--cratesio--github-release)). After `publish-docker` finishes, the image is at `ghcr.io/jchchiu/rust-varlap:<tag>`.
3. **Make the package public.** GitHub creates new packages as private, so `docker pull` and `apptainer pull` fail until you change it:
   - Go to github.com/jchchiu → **Packages** → **rust-varlap** → **Package settings**.
   - Under **Danger Zone**, use **Change visibility** and set it to **Public**.
   - On the same page, check that the repo is linked under **Manage Actions access**. The image's `org.opencontainers.image.source` label should link it automatically.

**Check it works:**
```bash
docker run --rm ghcr.io/jchchiu/rust-varlap:v0.1.0-alpha.6 rust-varlap --version
apptainer pull docker://ghcr.io/jchchiu/rust-varlap:v0.1.0-alpha.6
```

On prerelease tags (`-alpha`, `-beta`, `-rc`), the workflow pushes only the version tag. `:latest` is updated only on stable releases.

---

## crates.io

**Setup (once):**
1. Sign in at https://crates.io with your GitHub account, and verify your email under Account Settings. Publishing doesn't work until the email is verified.
2. **Account Settings → API Tokens → New Token**:
   - Name: `rust-varlap-github-actions`
   - Scopes: **publish-new** (needed for the first publish) and **publish-update**
   - Crates: `rust-varlap`
   - Expiry: up to you. Note the expiry date, because the release job fails once the token expires.
3. Add the token as a repo secret, either at **Settings → Secrets and variables → Actions → New repository secret** with the name `CARGO_REGISTRY_TOKEN`, or with:
   ```bash
   gh secret set CARGO_REGISTRY_TOKEN --repo jchchiu/rust-varlap
   ```

The `publish-crate` job then runs `cargo publish --locked` on each tag.

**Things to know:**
- **Publishing is permanent.** You can't delete a version or publish the same version twice. You can only yank a version (`cargo yank --version X`), which hides it from new installs. Run `cargo publish --dry-run --locked` before tagging if you've changed `Cargo.toml`.
- **Optional, no token needed:** after the first publish, you can switch to **Trusted Publishing**.
  - On crates.io, go to the crate → **Settings → Trusted Publishing** and add `jchchiu/rust-varlap` with workflow `release.yml`.
  - In `publish-crate`, replace the token with `rust-lang/crates-io-auth-action@v1` and add `permissions: id-token: write`.
  - You can then delete the secret.
- **Unused dev-dependency:** `Cargo.toml` has a dev-dependency `clippy = "0.0.302"`. That's an old placeholder crate, not the real clippy, which comes with rustup. It doesn't affect users, but you can remove it.

---

## Releasing (Docker + crates.io + GitHub release)

1. Bump the version in **both** `Cargo.toml` and `src/cli.rs`, for example to `0.1.0-alpha.6`. The `check-version` job fails if they differ from each other or from the tag.
2. Optional: run **Actions → Build & Release → Run workflow** first. This builds and tests everything but doesn't publish.
3. Commit, then tag and push:
   ```bash
   git tag v0.1.0-alpha.6
   git push origin v0.1.0-alpha.6
   ```
4. If every test passes, `release`, `publish-docker` and `publish-crate` run in parallel. They don't depend on each other, so if one fails the others can still publish.

**If only one publish job fails:**
- Fix the cause, for example a missing secret or private package permissions.
- Re-run only that job from the Actions page, using **Re-run failed jobs**.
- Don't re-tag. crates.io refuses a version it already has, and re-tagging would rebuild the GitHub release.

---

## Bioconda

Bioconda isn't automated from this repo. You add a recipe to [bioconda-recipes](https://github.com/bioconda/bioconda-recipes) once. After that, their bot opens a pull request to update the recipe each time you make a new GitHub release.

**Wait for a stable release first.** Conda versions can't contain `-`, so `0.1.0-alpha.5` would have to become something like `0.1.0a5`, and Bioconda reviewers usually push back on alpha releases. It's easier to submit once `v0.1.0` exists.

**Steps:**
1. Read the [contributor guide](https://bioconda.github.io/contributor/index.html). It covers the conda channel setup and the review process.
2. Fork `bioconda/bioconda-recipes` and create a branch `add-rust-varlap`.
3. Look at an existing Rust recipe that uses rust-htslib, such as `recipes/perbase/` (I believe it uses rust-htslib, but check its build requirements). Copy its compiler and htslib-related requirements, because those are the parts most likely to break.
4. Get the checksum of the release tarball:
   ```bash
   curl -sL https://github.com/jchchiu/rust-varlap/archive/refs/tags/v0.1.0.tar.gz | sha256sum
   ```
5. Create `recipes/rust-varlap/meta.yaml`, roughly like this:
   ```yaml
   {% set name = "rust-varlap" %}
   {% set version = "0.1.0" %}

   package:
     name: {{ name }}
     version: {{ version }}

   source:
     url: https://github.com/jchchiu/rust-varlap/archive/refs/tags/v{{ version }}.tar.gz
     sha256: <sha256 from step 4>

   build:
     number: 0
     run_exports:
       - {{ pin_subpackage(name, max_pin="x.x") }}

   requirements:
     build:
       - {{ compiler('rust') }}
       - {{ compiler('c') }}
       - cargo-bundle-licenses
       - clangdev        # bindgen in hts-sys needs libclang
       - cmake
       - make
       - pkg-config
     host:
       - zlib
       - bzip2
       - xz
       - libcurl
       - openssl

   test:
     commands:
       - rust-varlap --version
       - rust-varlap --help

   about:
     home: https://github.com/jchchiu/rust-varlap
     license: MIT
     license_file:
       - LICENSE
       - THIRDPARTY.yml
     summary: Quality control tool for genetic variants
     doc_url: https://jchchiu.github.io/rust-varlap/
     dev_url: https://github.com/jchchiu/rust-varlap

   extra:
     additional-platforms:
       - linux-aarch64
       - osx-arm64
   ```
6. Create `recipes/rust-varlap/build.sh`:
   ```bash
   #!/bin/bash
   set -euo pipefail
   cargo-bundle-licenses --format yaml --output THIRDPARTY.yml
   cargo install --no-track --locked --verbose --root "${PREFIX}" --path .
   ```
7. Open a pull request to `bioconda/bioconda-recipes`. Their CI builds and tests the recipe on every platform. Once it passes and a reviewer approves, comment `@BiocondaBot please add label` to request the merge label.
8. After the merge:
   - `conda install -c bioconda rust-varlap` works.
   - A BioContainers Docker image (`quay.io/biocontainers/rust-varlap`) is built automatically. That's the image nf-core pipelines use.

**Afterwards:**
- **Version bumps:** each new GitHub release gets an automatic version-bump pull request from the Bioconda bot. You review and merge it, but you don't need to write anything.
- **Docs:** when it's live, remove the "Coming soon" note in `docs/src/1_getting_started/installation.md` and in `README.md`.

---

## Checklist

- [ ] Commit the release workflow, Dockerfile, `Cargo.toml`, docs workflow and README changes
- [ ] Add the `CARGO_REGISTRY_TOKEN` secret
- [ ] Run the release workflow manually (workflow_dispatch) as a dry run
- [ ] Bump the version in `Cargo.toml` and `src/cli.rs`, then tag and push
- [ ] Make the ghcr.io package public
- [ ] Optional: switch crates.io to Trusted Publishing
- [ ] After `v0.1.0` (stable): submit the Bioconda recipe
- [ ] After Bioconda goes live: update the installation docs and README
