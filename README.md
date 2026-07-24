# oCIS Workflows Docker Image

[![Docker Pulls](https://img.shields.io/docker/pulls/owncloud/ocis-workflows.svg)](https://hub.docker.com/r/owncloud/ocis-workflows)
[![License: Apache-2.0](https://img.shields.io/github/license/owncloud-docker/ocis-workflows)](https://github.com/owncloud-docker/ocis-workflows/blob/main/LICENSE)

Docker image for [oCIS Workflows](https://github.com/owncloud/ocis-workflows) — an
AI-powered file workflow automation extension for
[ownCloud Infinite Scale (oCIS)](https://github.com/owncloud/ocis). This repo builds and
publishes the image; the application source lives in
[owncloud/ocis-workflows](https://github.com/owncloud/ocis-workflows).

## Quick Start

```bash
docker run --rm \
  -p 9105:9105 \
  -p 9109:9109 \
  -e WORKFLOWS_OCIS_URL=https://ocis.example.com \
  -e WORKFLOWS_ALLOWED_ORIGIN=https://ocis.example.com \
  owncloud/ocis-workflows:latest
```

See [owncloud/ocis-workflows](https://github.com/owncloud/ocis-workflows#readme) for the
full list of `WORKFLOWS_*` environment variables.

## Supported Tags

`owncloud/ocis-workflows` has no upstream release tags yet, so this image currently tracks
the unreleased `main` branch of `ocis-workflows` directly (a rolling build), rather than
a pinned-version matrix.

| Tag | Meaning |
|-----|---------|
| `latest` | Most recent build of the `main` branch |
| `YYYYMMDD` | Immutable build for a specific day (e.g. `20260602`) |
| `sha-<short>` | Build of a specific `ocis-workflows` `main` commit (e.g. `sha-a1b2c3d`) |

> Once `ocis-workflows` starts cutting semver releases, this repo's
> `.github/workflows/main.yml` should be extended with a version matrix (see
> [owncloud-docker/ocis](https://github.com/owncloud-docker/ocis)'s `main.yml` for the
> pattern) to publish pinned, immutable version tags.

```bash
docker pull owncloud/ocis-workflows:latest
```

## Two Deployables, One Image

`ocis-workflows` ships a Go backend sidecar and a Vue frontend extension for ownCloud Web.
This image bundles both from a single build so they always ship in lockstep:

- The backend binary is the image's `ENTRYPOINT` (`CMD ["server"]`) — run the container
  directly to start the API/automation sidecar.
- The built frontend (`pnpm build` output) is baked in at `/web/apps/workflows`. This
  image does **not** serve it — oCIS's own web server does. Extract it into your oCIS
  deployment's web apps directory (`WEB_ASSET_APPS_PATH`), for example via an
  `initContainer` that just copies the path out to a shared volume:

  ```yaml
  initContainers:
    - name: workflows-web-assets
      image: owncloud/ocis-workflows:latest
      command: ["cp", "-r", "/web/apps/workflows", "/web-apps/workflows"]
      volumeMounts:
        - name: web-apps
          mountPath: /web-apps
  ```

  This mirrors how the app repo's `docker-compose.yml` mounts `./frontend/dist` straight
  into the `ocis` container for local development.

## Volumes

| Path | Purpose |
|------|---------|
| `/data` | Local operational database (`WORKFLOWS_DB_PATH`) |

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `9105` | TCP | Public API (`WORKFLOWS_HTTP_ADDR`) |
| `9109` | TCP | Health/debug endpoint, `/healthz` (`WORKFLOWS_DEBUG_ADDR`) |

## Build Arguments

| ARG | Default | Purpose |
|-----|---------|---------|
| `GIT_REF` | `main` | `ocis-workflows` git ref (branch or tag) to clone and build |
| `GIT_SHA` | `""` | Optional exact commit to check out after cloning `GIT_REF`. Pins the build to a resolved commit and busts the clone-layer cache when it changes, keeping rolling builds fresh |
| `VERSION` | `""` | Version string embedded in OCI labels |
| `REVISION` | `""` | Git SHA embedded in OCI labels |
| `TARGETARCH` | set by buildx | Target architecture (`amd64`, `arm64`) |

## Building

The image is built entirely from source via a three-stage `Dockerfile.multiarch`:

**`frontend-builder`** — clones `ocis-workflows` at `${GIT_REF}` and builds the Vue
frontend extension (`pnpm build`) from `frontend/`.

**`go-builder`** — compiles the backend binary from `backend/cmd/workflows` with
`CGO_ENABLED=0` for the target architecture.

**Runtime** — minimal Alpine image with the compiled binary and the built frontend
assets copied in. Runs as a non-root user. The stage runs `apk upgrade` to refresh all
installed OS packages to the latest available Alpine patch releases at build time, so
security fixes are picked up immediately rather than waiting for a base-image tag bump.

```bash
docker buildx build -f Dockerfile.multiarch --build-arg GIT_REF=main -t ocis-workflows:test .
```
