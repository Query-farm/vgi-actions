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

## Binary / artifact release workflows

Three language-specific reusable workflows attach a worker's release artifacts to
the GitHub Release for a `vX.Y.Z` tag — each with a **SHA256**, a keyless
**cosign signature** (`.cosign.bundle`), and a **SLSA build-provenance**
attestation. All are `workflow_call`-only; the **caller gates on its own test
suite** (`ci.yml` as a `needs:` job) and triggers on tags.

| Workflow | For | Artifacts |
| --- | --- | --- |
| `rust-release.yml` | Rust (cargo) workers | one `.tar.gz` per DuckDB platform |
| `go-release.yml` | Go workers | one `.tar.gz` per DuckDB platform (cross-compiled) |
| `java-release.yml` | Java (JVM) workers | one platform-independent `.jar` |

The binary workflows name assets `<asset_prefix>-<tag>-<duckdb_platform>.tar.gz`
across the five DuckDB platforms (`linux_amd64`, `linux_arm64`, `osx_amd64`,
`osx_arm64`, `windows_amd64`) — `.tar.gz` on **every** platform (Windows
included; the vgi client only reads `.tar.gz`). Java publishes a single
`<asset_prefix>-<tag>.jar`.

Signing runs in **these** workflows, so the keyless cert identity is the
vgi-actions workflow (`…/rust-release.yml`, `…/go-release.yml`,
`…/java-release.yml`), the same model as the signed images — not the caller's
`release.yml`. Verify, e.g.:

```sh
cosign verify-blob \
  --bundle vgi-units-v0.2.0-linux_amd64.tar.gz.cosign.bundle \
  --certificate-identity-regexp '^https://github\.com/Query-farm/vgi-actions/\.github/workflows/rust-release\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  vgi-units-v0.2.0-linux_amd64.tar.gz

gh attestation verify vgi-units-v0.2.0-linux_amd64.tar.gz \
  --repo Query-farm/vgi-units --signer-repo Query-farm/vgi-actions
```

### `rust-release.yml`

Builds the `bin` for every DuckDB platform. macOS builds on **Apple Silicon**
(`macos-15`); the Intel (`osx_amd64`) binary is **cross-compiled** (`macos-13` is
deprecated). Cargo downloads are hardened against CDN flakes (`CARGO_NET_RETRY`,
`CARGO_HTTP_MULTIPLEXING=false`).

```yaml
  release:
    needs: [ci]
    permissions: { contents: write, id-token: write, attestations: write }
    uses: Query-farm/vgi-actions/.github/workflows/rust-release.yml@v1
    secrets: inherit
    with:
      bin: units-worker            # cargo bin (executable inside the archive)
      asset_prefix: vgi-units      # optional; defaults to the repo name
      version_check_cmd: ci/check-version.sh   # optional
```

Inputs: `bin` (required), `asset_prefix` (default repo name), `version_check_cmd`
(default skip), `include` (default `README.md,LICENSE`).

### `go-release.yml`

Cross-compiles all five platforms on one linux runner via `GOOS`/`GOARCH`
(`CGO_ENABLED=0` by default). The Windows binary is `<bin>.exe` inside the
`.tar.gz`.

```yaml
  release:
    needs: [ci]
    permissions: { contents: write, id-token: write, attestations: write }
    uses: Query-farm/vgi-actions/.github/workflows/go-release.yml@v1
    secrets: inherit
    with:
      bin: vgi-grpc                # output binary name; defaults to the repo name
      package: ./cmd/worker        # go build target; default "."
      ldflags: "-s -w -X main.version=${TAG}"   # ${TAG} -> release tag
```

Inputs: `bin` (default repo name), `package` (default `.`), `asset_prefix`
(default repo name), `version_check_cmd`, `ldflags` (default `-s -w`),
`go_version` (default `stable`), `include`, `cgo` (default `false`).

### `java-release.yml`

Builds one runnable fat jar (no platform matrix). The `jar` glob must point at
the **uber/shaded** jar (not the thin jar).

```yaml
  release:
    needs: [ci]
    permissions: { contents: write, id-token: write, attestations: write }
    uses: Query-farm/vgi-actions/.github/workflows/java-release.yml@v1
    secrets: inherit
    with:
      jar: target/*-jar-with-dependencies.jar   # required: the runnable jar
      build_cmd: "mvn -B -ntp -DskipTests package"
      java_version: "21"
```

Inputs: `jar` (required), `build_cmd` (default `mvn -B -ntp -DskipTests
package`), `asset_prefix` (default repo name), `version_check_cmd`,
`java_version` (default `21`), `java_distribution` (default `temurin`), `cache` (default none; set `gradle` for Gradle projects).
