# vgi-actions

Reusable GitHub Actions / workflows for [VGI](https://query.farm) worker repos,
maintained by [Query.Farm](https://query.farm).

## `docker-publish.yml` — multi-arch container build & publish

A reusable workflow that builds a worker's container image for **linux/amd64 +
linux/arm64**, tests each arch before pushing, publishes to **ghcr.io** by
digest under a multi-arch manifest, and **cosign-signs** it (keyless).

Per arch it runs an **import smoke** and a **dual-transport `/health` boot**;
on amd64 it additionally runs an optional caller-provided **`image_test`** for
the full suite against the container. Tags follow the caller's git context:
`vX.Y.Z` → `:X.Y.Z` + `:X.Y` + `:latest`, default-branch push → `:edge`, plus
`:sha`.

### Usage

This is `workflow_call`-only. The **caller gates on its own test suite** (run
`ci.yml` as a `needs:` job), then calls this workflow:

```yaml
name: Publish image to ghcr.io
on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: ./.github/workflows/ci.yml          # gate: the repo's own suite
  publish:
    needs: [ci]
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    uses: Query-farm/vgi-actions/.github/workflows/docker-publish.yml@v1
    secrets: inherit
    with:
      image_name: vgi-sklearn
      smoke_import: "vgi_sklearn, sklearn, numpy, scipy"
      http_run_args: "-e VGI_SIGNING_KEY=dev"
      version_check_cmd: "ci/check-version.sh"
      # Optional: full sqllogictest/pytest suite against the built image.
      image_test: |
        # $CITEST_IMAGE is the loaded <image>:citest-amd64 ref.
        ...
```

### Inputs

| Input | Required | Default | Purpose |
| --- | --- | --- | --- |
| `image_name` | yes | — | Image name under `ghcr.io/<owner>/`. |
| `context` | no | `.` | Docker build context. |
| `dockerfile` | no | `Dockerfile` | Path to the Dockerfile. |
| `smoke_import` | no | `""` | Comma-separated modules to `import` in an in-container smoke (both arches). Empty = skip. |
| `health_port` | no | `8000` | Port the HTTP transport serves `/health` on. |
| `http_run_args` | no | `""` | Extra `docker run` args for the HTTP boot smoke (e.g. `-e VGI_SIGNING_KEY=dev`). |
| `version_check_cmd` | no | `""` | Command run on a version-tag push with the tag as `$1` (e.g. `ci/check-version.sh`). Empty = skip. |
| `image_test` | no | `""` | Shell run on amd64 after build for the full suite; `$CITEST_IMAGE` = loaded image ref. Empty = smoke only. |

### Image contract

The workflow expects the worker's Dockerfile to produce an image that:

- defaults to the **HTTP transport** (serving `/health` on `health_port`) and
  accepts a `stdio` argument for the stdio transport (see the shared
  `docker-entrypoint.sh` pattern in the worker repos);
- accepts `--build-arg VERSION=` and `--build-arg GIT_COMMIT=`.

Notes:

- A **reusable workflow cannot run the caller's `ci.yml`** itself (a `./`
  workflow path resolves inside *this* repo), so gating lives in the caller.
- Multi-arch is fixed to amd64 + arm64 on native runners; that is the supported
  baseline.

## `binary-release.yml` — cross-platform binary build & GitHub Release

A reusable workflow for **binary-driven (Rust) workers**: it builds the worker
binary for every supported platform and attaches the archives to the GitHub
Release for a `vX.Y.Z` tag, each with a **SHA256**, a keyless **cosign
signature** (`.cosign.bundle`), and a **SLSA build-provenance** attestation.

Standardized so every worker ships the same shape:

- one **`.tar.gz` per DuckDB platform** (`linux_amd64`, `linux_arm64`,
  `osx_amd64`, `osx_arm64`, `windows_amd64`) — Windows ships `.tar.gz` too (not
  `.zip`), since the vgi client only reads `.tar.gz`. Assets are named
  `<asset_prefix>-<tag>-<duckdb_platform>.tar.gz`.
- macOS builds on **Apple Silicon** (`macos-15`); the Intel (`osx_amd64`) binary
  is **cross-compiled** (the last Intel runner image, `macos-13`, is
  scarce/deprecated). Every other target builds native.

### Usage

`workflow_call`-only; the **caller gates on its own test suite**, then calls it:

```yaml
name: Release binaries
on:
  push:
    tags: ['v*.*.*']
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: ./.github/workflows/ci.yml          # gate: the repo's own suite
  release:
    needs: [ci]
    permissions:
      contents: write       # create Release + upload assets
      id-token: write       # keyless cosign + provenance (sigstore OIDC)
      attestations: write   # SLSA build-provenance
    uses: Query-farm/vgi-actions/.github/workflows/binary-release.yml@v1
    secrets: inherit
    with:
      bin: units-worker            # cargo bin to build (executable in the archive)
      asset_prefix: vgi-units      # optional; defaults to the repo name
      version_check_cmd: ci/check-version.sh
```

### Inputs

| Input | Required | Default | Purpose |
| --- | --- | --- | --- |
| `bin` | yes | — | Cargo bin name to build and package (the executable inside each archive). |
| `asset_prefix` | no | repo name | Archive base-name prefix → `<asset_prefix>-<tag>-<platform>.tar.gz`. |
| `version_check_cmd` | no | `""` | Command run on a version-tag push with the tag as `$1` (e.g. `ci/check-version.sh`). Empty = skip. |
| `include` | no | `README.md,LICENSE` | Comma-separated extra files to include in each archive. |

### Verifying a release binary

Signing runs in **this** workflow, so the keyless identity is
`Query-farm/vgi-actions/.github/workflows/binary-release.yml` (same model as the
signed images), not the caller's `release.yml`:

```sh
cosign verify-blob \
  --bundle vgi-units-v0.1.3-linux_amd64.tar.gz.cosign.bundle \
  --certificate-identity-regexp '^https://github\.com/Query-farm/vgi-actions/\.github/workflows/binary-release\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  vgi-units-v0.1.3-linux_amd64.tar.gz

# Provenance (verify against the caller repo; the signer is vgi-actions):
gh attestation verify vgi-units-v0.1.3-linux_amd64.tar.gz \
  --repo Query-farm/vgi-units \
  --signer-repo Query-farm/vgi-actions
```

### Contract

The workflow expects a Cargo workspace whose `bin` target builds with
`cargo build --release --locked`. A `version_check_cmd` (if given) is a
repo-local script taking the tag as `$1` and exiting non-zero on mismatch.
