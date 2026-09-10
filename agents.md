# agents.md — ocis-workflows

## Repository Overview

This repository builds the official **oCIS Workflows** Docker image
(`owncloud/ocis-workflows` on Docker Hub). It does not contain the ocis-workflows
source code — it builds it **from source** via a multi-stage Dockerfile and ships a
minimal Alpine runtime image. Images are multi-architecture and built via GitHub
Actions.

Unlike the other image repos in this organisation, one image carries **two
deployables**: the Go backend sidecar and the Vue web extension for ownCloud Web.
They are built from a single upstream checkout so they always ship in lockstep.

- **Classification:** Docker image build (from source)
- **Activity Status:** Active
- **License:** Apache-2.0 (with a BSD-2-Clause third-party dependency — see below)
- **Language:** Dockerfile

## Architecture & Key Paths

- `Dockerfile.multiarch` — three-stage build (build context is the repo root):
  - **frontend-builder** (`node:24-alpine`) — clones `owncloud/ocis-workflows` at
    `${GIT_REF}` (optionally pinned to `${GIT_SHA}`) and builds the Vue frontend
    extension with `pnpm build` from `frontend/`.
  - **go-builder** (`golang:1.26-alpine`) — compiles `backend/cmd/workflows` with
    `CGO_ENABLED=0` for `${TARGETARCH}`.
  - **runtime** (`alpine`) — runs `apk upgrade` so OS security fixes are picked up
    at build time, adds a non-root `workflows-user` (uid 1000), and copies in the
    binary (`/usr/local/bin/app`, the `ENTRYPOINT`, `CMD ["server"]`) plus the built
    frontend assets (`/web/apps/workflows`).
- `.github/workflows/main.yml` — **active** CI: rolling build of upstream `main`
- `.github/workflows/lint-pr-title.yml` — Conventional-Commit PR-title enforcement
- `.github/dependabot.yml` — weekly GitHub Actions and Docker base-image dependency updates
- `.github/CODEOWNERS` — review ownership
- `.editorconfig` — formatting rules (2-space indent, LF, trailing newline)
- `.trivyignore` — accepted-CVE exclusions for the Trivy scan
- `LICENSE` — Apache-2.0
- `NOTICE.md`, `LICENSES/BSD-2-Clause.txt` — third-party attribution: the shipped
  binary links `unraid/apprise-go` (BSD-2-Clause), used by `backend/pkg/notify`

There is **no `CHANGELOG.md`** in this repository.

## The image is not self-serving

The backend binary is the entrypoint — run the container to start the API/automation
sidecar. The frontend at `/web/apps/workflows` is **not** served by this image; oCIS's
own web server serves it. Extract that path into the oCIS deployment's
`WEB_ASSET_APPS_PATH`, e.g. via an `initContainer` that copies it to a shared volume.
See the [README](README.md) for the manifest.

## Build & CI

CI (`main.yml`) calls the reusable `docker-build-native.yml` workflow hosted in
[`owncloud-docker/ubuntu`](https://github.com/owncloud-docker/ubuntu) — the per-arch
native build plus manifest merge, because building from source cross-platform is
expensive.

- **No version matrix.** Upstream `ocis-workflows` has no release tags yet, so the
  `prepare` job resolves `owncloud/ocis-workflows` `main` HEAD and the build tracks it
  (a rolling-style build). Tags published: `latest`, `YYYYMMDD`, `sha-<short>`. Once
  upstream cuts semver releases, extend `build` with a matrix following
  [`owncloud-docker/ocis`](https://github.com/owncloud-docker/ocis)'s `main.yml`.
- `GIT_SHA` is passed the resolved full SHA, which busts the clone layer's cache so
  the rolling build stays fresh.
- Weekly rebuild on `0 0 * * 0`, plus `workflow_dispatch`.
- Smoke test: start `/usr/local/bin/app server` and poll
  `http://localhost:9109/healthz`.
- Trivy vulnerability scan (`.trivyignore`), `exit-code: 1` — a HIGH/CRITICAL finding
  fails the job, so nothing vulnerable is pushed.
- On non-PR events: push to Docker Hub and sync the README as the image description.

To build locally:

```bash
docker buildx build -f Dockerfile.multiarch --build-arg GIT_REF=main -t ocis-workflows:test .
```

The image exposes ports `9105` (API) and `9109` (health/debug), and uses volume `/data`.

## Development Conventions

- **No CHANGELOG** — do not create one.
- Conventional-Commit PR titles, enforced by `lint-pr-title.yml`.
- `.editorconfig` governs formatting; `LICENSE` and `LICENSES/*` are exempt from the
  indent rules.
- GitHub Actions are pinned to full commit SHAs.
- Workflows declare a least-privilege `permissions:` block.
- Bug reports for the application itself go upstream to
  [`owncloud/ocis-workflows`](https://github.com/owncloud/ocis-workflows); this repo
  tracks only the Docker packaging.
- Adding a non-Apache-2.0 third-party dependency to the shipped binary means updating
  `NOTICE.md` and adding its licence text under `LICENSES/`.

## OSPO Policy Constraints

### GitHub Actions
- **Only** use actions owned by `owncloud`, created by GitHub (`actions/*`),
  verified on the GitHub Marketplace, or verified by the ownCloud Maintainers.
- Pin all actions to their full commit SHA (not tags): `uses: actions/checkout@<SHA> # vX.Y.Z`.
- Never introduce actions from unverified third parties.

### Dependency Management
- Dependabot covers both GitHub Actions and Docker base-image updates for this
  repo (`.github/dependabot.yml`) — Renovate is no longer used here.
- Unlike the old Renovate setup, Dependabot does not auto-merge digest-only
  updates and doesn't restrict Node bumps to LTS (even-major) releases. Review
  each Docker/Actions PR before merging, including checking whether a Node
  bump lands on an odd (non-LTS) major.
- Review and merge dependency PRs as part of regular maintenance. A stale
  `golang:*-alpine` digest surfaces as Go `stdlib` CVEs failing the Trivy gate; the
  fix is bumping the digest, not adding `.trivyignore` entries.

### Git Workflow
- **Rebase policy**: Always rebase; never create merge commits.
- **Signed commits**: All commits **must** be PGP/GPG signed (`git commit -S`).
- **DCO sign-off**: Every commit needs a `Signed-off-by` line (`git commit -s`).
- **Conventional Commits & Squash Merge**: PR titles must follow
  [Conventional Commits](https://www.conventionalcommits.org/); the PR title
  becomes the squash-merge commit message and is enforced by CI.

## Context for AI Agents

- This is a Docker-image build repo that compiles ocis-workflows from source — not the
  application codebase. Application changes belong upstream.
- One image, two deployables. A change to how the frontend assets are laid out affects
  consumers' `initContainer` copy step, so it is a breaking change for deployments.
- The build tracks upstream `main`; there is deliberately no version matrix yet.
- The README is published verbatim as the Docker Hub image description — keep it
  accurate and self-contained.
- License is **Apache-2.0**, the OSPO's ecosystem-wide target; no relicensing needed.
  The BSD-2-Clause dependency is attribution-only and does not change that.
