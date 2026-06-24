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
