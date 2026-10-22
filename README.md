# devcontainer-images

Golden dev-environment images for workstation devpods: a `base` image carrying the
common tool belt, and thin language flavours layered on top (`dotnet`). Images are
deterministic Docker builds from public base layers (`mcr.microsoft.com/devcontainers/base`
and `ghcr.io/jdx/mise`), with every tool version pinned in `mise.toml` and locked in
`mise.lock`, so anyone can rebuild them byte-for-byte from this repository. Personal
config never bakes in; it lands at devpod start-up via the dotfiles hook.

## Layout

- `base/` — base Dockerfile + mise config and lockfile (the pinned tool belt)
- `dotnet/` — .NET flavour, `FROM` base via the `BASE` build-arg; pins the .NET 8 and 10 SDKs
- `tasks/` — build and test entry points, run via `mise run <task>`
- `.github/workflows/build.yaml` — CI build and publish
- `renovate.json` — version-bump PRs for the pins
- `.devcontainer/` — the devpod this repo is worked from

Bumping one tool re-runs only the `mise install` layer; OS layers stay cached, so a
version bump ships as one small content-addressed blob.

## Local loop

The tasks are rungs of one verification ladder. `mise tasks` lists them.

- **Rung 1, no Docker:** `test-config` resolves every `tool@version` in each flavour's
  `mise.toml` against the lockfile, so a bad pin fails before any image is built.
- **Rung 2, local images:** `build-base` and `build-dotnet` build `devcontainer-base:local`
  and `devcontainer-dotnet:local`; `test-base` and `test-dotnet` run each image as the
  `vscode` user and assert the belt and SDKs are present and runnable.
- `verify` runs the whole ladder.
- **Rung 3, publish:** see Publishing below.

## Publishing

Images are published to the private Harbor `devcontainer-images` project as immutable
date tags (`YYYY.M.DD`, no leading zero on the month) with an SBOM attestation, and a
moving `stable` tag points at the most recent date tag that passed the verification
ladder. Consumers pin `stable` by digest. Pushes authenticate as the project-scoped robot
account only, never an admin user.

Pushes have to originate inside the LAN. Harbor is fronted by a Cloudflare tunnel whose
per-request body cap sits far below an image layer, so anything outside the LAN can pull,
browse and call the API but cannot push. That gives two publishers:

- **Rung 3, workstation:** `publish` runs the ladder, then builds both flavours with
  `buildx` on a `docker-container` builder (the default driver cannot push attestations)
  and pushes the date tags. `promote-stable [TAG]` re-tags a verified date tag as
  `stable` by digest, so nothing is rebuilt and the attestations ride along. Both tasks
  read `HARBOR_ROBOT_USERNAME` / `HARBOR_ROBOT_SECRET` from the environment (the secret is
  the Key Vault entry `harbor-gha-robot-creds`), refuse any non-robot username, refuse to
  run if the registry does not resolve to a private address, and log in under a throwaway
  `DOCKER_CONFIG` so the workstation's own registry login is left alone.
- **CI, `build.yaml`:** the same build on a runner, plus a GitHub release with generated
  notes. It runs by hand only until an in-LAN runner exists; GitHub-hosted runners sit
  outside the LAN and fail at the login step.

The Harbor project is private. Nothing here needs it to be public: the Dockerfiles and
lockfiles are the reproducible artefact.

## Prune safety

The local Docker image cache is the standing fallback when the registry is unreachable:
the last-known-good `devcontainer-base:local` and `devcontainer-dotnet:local` images let a
devpod start with no registry at all. A blanket `docker system prune -a` (or
`docker image prune -a`) evicts every image not currently used by a container, including
those fallbacks, so `-a` prunes are the one habit that silently deletes the safety net.

When reclaiming space:

- `docker image prune` (no `-a`) removes dangling build residue only; tagged flavour
  images are never touched. This is the safe default.
- `docker system prune` (no `-a`) follows the same rule: stopped containers, networks and
  dangling images, but no tagged images.
- If a deeper clean is genuinely needed, rebuild the flavours first (`mise run build-base`,
  `mise run build-dotnet`) so a last-known-good image exists before anything is evicted.
