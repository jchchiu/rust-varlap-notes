# Release workflow update

Changes to `.github/workflows/release.yml` on branch `add-readthedocs-yaml`, 2026-11-08. The goal was for a `v*` tag to publish every installation option in `docs/src/1_getting_started/installation.md`.

**Status:** the changes are in the working tree and are **not committed**. The workflow has **not been run**. No YAML linter could be installed locally (actionlint was blocked), so the YAML hasn't been machine-checked. The first real test should be a manual **workflow_dispatch** run, which builds and tests everything but skips all publishing.

## Files changed

| File | Change |
|---|---|
| `.github/workflows/release.yml` | Rewritten: 4-platform binary matrix, Docker test and publish, crates.io publish, version check, CLI fix |
| `.github/docker/Dockerfile` | New. Ubuntu 24.04 runtime image that copies in a pre-built binary |
| `Cargo.toml` | Added `exclude` so the crate stays under crates.io's 10 MB limit |

## Install options covered

| Install option in the docs | What the workflow does |
|---|---|
| Pre-built binaries | Builds 4 targets on native runners. They go on the release as `rust-varlap_${TAG}_<target>`, the names the docs use. |
| Docker | Pushes a combined amd64 + arm64 image to `ghcr.io/jchchiu/rust-varlap:${TAG}`. `:latest` moves only for full releases, not alpha/beta/rc. |
| Apptainer, pulled from the registry | Uses the ghcr.io image above. |
| Apptainer, `.sif` download | Same as before: built from the linux-x86_64 binary and attached to the release. |
| crates.io | `cargo publish --locked`. |
| Bioconda | **Not added.** A Bioconda recipe lives in the separate `bioconda-recipes` repo, and their bot picks up new GitHub releases from there. |
| Build from source / `cargo install --git --tag` | Needs only the pushed tag, so nothing to add. |

## Job layout

| Job | Runs on | Needs | Notes |
|---|---|---|---|
| `check-version` | ubuntu-24.04 | — | `Cargo.toml` and `src/cli.rs` must match. On a real push, the tag must also equal `v<version>`. |
| `build-binary` (matrix) | see below | — | Uploads artifact `rust-varlap-<target>` |
| `test-binary` (matrix) | same runner as build | `build-binary` | Smoke test and regression test against `python.chrM_heavy_stress.snv.csv` |
| `build-sif` | ubuntu-24.04 | `build-binary` | Unchanged apart from the artifact name |
| `test-sif` | ubuntu-24.04 | `build-sif` | Regression test now uses `/test_data/...` paths inside the bind mount |
| `test-docker` | ubuntu-24.04 | `build-binary` | Builds an amd64 image locally (`load: true`), runs the regression test, pushes nothing |
| `release` | ubuntu-24.04 | all tests + `check-version` | Push events only. Renames the binaries and creates the GitHub release |
| `publish-docker` | ubuntu-24.04 | all tests + `check-version` | Push events only. Needs QEMU for the arm64 `apt-get` layer. `packages: write` |
| `publish-crate` | ubuntu-24.04 | all tests + `check-version` | Push events only. Needs the `CARGO_REGISTRY_TOKEN` secret |

Binary matrix:

| Target | Runner |
|---|---|
| `linux-x86_64` | `ubuntu-24.04` |
| `linux-aarch64` | `ubuntu-24.04-arm` |
| `macos-aarch64` | `macos-15` |
| `macos-x86_64` | `macos-15-intel` |

The three publish jobs run in parallel after every test passes. They don't depend on each other, so if one fails the others can still publish. For example, the GitHub release could go out while crates.io fails.

## Fixes along the way

- **Broken tests:** the binary and `.sif` test steps still used the old flat CLI (`--variants … --varclass snv`). They would fail against the `loci`/`region` subcommands. They now use `loci -v … -r … -c snv -o …`.
- **Crate size:** without `exclude`, `cargo package` would include `test_data/chrM_heavy_stress.sorted.bam` (14.7 MB), which is over crates.io's 10 MB limit. After the change, `cargo package --no-verify` gives 20 files and 38 KB. Excluded: `test_data/`, `docs/`, `.github/`, `.readthedocs.yaml` and `*.code-workspace`.
- **Linux deps:** added `clang` and `libclang-dev` to the build dependencies, because bindgen needs libclang and the arm runner image might not have it.

## macOS portability

macOS binaries could end up needing Homebrew libraries on the user's machine. To avoid this:

- zlib, bzip2 and libcurl come from macOS itself.
- OpenSSL is linked statically: `OPENSSL_STATIC=1` and `OPENSSL_DIR=$(brew --prefix openssl@3)`.
- liblzma is linked statically: `LZMA_API_STATIC=1` makes `lzma-sys` build its bundled copy.
- A check step runs `otool -L` and fails the build if the binary still links to anything under `/opt/homebrew` or `/usr/local`.

This is the part most likely to need tweaking on the first run. It was written without access to a macOS machine.

Linux binaries are still linked dynamically against the distro's libcurl, libssl and other libraries, as before. Users need the runtime packages that the docs and the container images list.

## Docker image

`.github/docker/Dockerfile`:

- Base `ubuntu:24.04`. Runtime packages: `ca-certificates`, `libcurl4t64`, `zlib1g`, `libbz2-1.0` and `liblzma5`. `libssl3t64` comes in as a dependency of libcurl.
- `COPY ${TARGETARCH}/rust-varlap`. The build context must contain `amd64/rust-varlap` and `arm64/rust-varlap`.
- `WORKDIR /data`, `CMD ["rust-varlap", "--help"]`. There's no `ENTRYPOINT`, so the command is passed explicitly, as the docs show: `docker run … image rust-varlap --version`.
- OCI labels (`source`, `version`, `title`, `licenses`) are set by `build-push-action`. The `source` label links the ghcr.io package to the repo.

## To do before the first tagged release

1. Add a `CARGO_REGISTRY_TOKEN` repository secret.
2. After the first push, make the ghcr.io package public. GitHub creates it private, and while it is private `docker pull` / `apptainer pull` need a login.
3. If `0.1.0-alpha.5` is already on crates.io, re-tagging it will fail `publish-crate`, because crates.io won't accept the same version twice. Bump the version in both `Cargo.toml` and `src/cli.rs`.
4. Run the workflow manually (workflow_dispatch) first to check the build and test jobs.
5. Commit the three changed files.

## Loose ends (not changed)

- `.readthedocs.yaml` still configures Sphinx (`docs/conf.py`), but the docs are now mdBook.
- The repo's `CLAUDE.md` still says the release builds "the Linux binary and an Apptainer `.sif`". It should list the 4 binaries, Docker and crates.io.
- `installation.md` notes that an image pulled from the registry may lack a runscript. That's still true: the Docker image has a `CMD` but no `ENTRYPOINT`, so `./rust-varlap.sif --version` won't work on a pulled image, and `apptainer exec` is needed.
- The Dockerfile drops `libdeflate0`, because rust-htslib's `libdeflate` feature isn't enabled. The `.sif` definition still installs it. That's harmless, but the two are inconsistent.
