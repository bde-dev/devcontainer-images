# devcontainer-images

Golden dev-environment images for workstation devpods: a `base` image carrying the
common tool belt, and thin language flavours layered on top (`dotnet`). Images are
deterministic Docker builds from public base layers (`mcr.microsoft.com/devcontainers/base`
and `ghcr.io/jdx/mise`), with every tool version pinned in `mise.toml` and locked in
`mise.lock`, so anyone can rebuild them byte-for-byte from this repository. Personal
config never bakes in; it lands at devpod start-up via the dotfiles hook.

## Layout

- `base/` — base Dockerfile + mise config and lockfile (the pinned tool belt)
- `dotnet/` — .NET flavour, built on the base version its Dockerfile pins in `ARG BASE`; pins the .NET 8 and 10 SDKs
- `tasks/` — build, test and publish entry points, run via `mise run <task>`
- `scripts/` — the release logic the tasks and CI share: which images changed, which version comes next
- `.github/workflows/verify.yaml` — CI check: the verification ladder on every pull request
- `.github/workflows/build.yaml` — CI release
- `renovate.json` — version-bump PRs for the pins
- `.devcontainer/` — the devpod this repo is worked from

Bumping one tool re-runs only the `mise install` layer; OS layers stay cached, so a
version bump ships as one small content-addressed blob.

## Local loop

The tasks are rungs of one verification ladder. `mise tasks` lists them.

- **Rung 1, no Docker:** `test-config` resolves every `tool@version` in each image's
  `mise.toml` against the lockfile, so a bad pin fails before any image is built.
  `test-version` and `test-release` check the release logic: the version a release would
  pick, and which images a release covers. `test-pins` checks that every image names the
  version of what it is built on. `test-workflows` checks that no workflow triggered by a
  pull request can reach a secret, a registry login or the publish route (see Pull requests).
- **Rung 2, local images:** `build-base` and `build-dotnet` build `devcontainer-base:local`
  and `devcontainer-dotnet:local`; `test-base` and `test-dotnet` run each image as the
  `vscode` user and assert the belt and SDKs are present and runnable. Here a flavour is
  built on the base from the working tree, not the version it pins, so a base change is
  tried against every flavour before either is released.
- `verify` runs the whole ladder.
- **Rung 3, publish:** see Publishing below.

## Pull requests

`verify.yaml` runs `mise run verify`, the whole ladder above, on every pull request
against `main`, on a GitHub-hosted runner, as one job named `verify`. That job is the check
to require on `main`: a pin that does not resolve, an image that does not build or a belt
tool that does not run turns it red before the change can merge. A required check has to
report on every pull request, so the workflow has no path filter and the job no condition
that could skip it. A change that touches no image still runs the ladder, because the ladder
also covers the release scripts and the workflows.

The ladder needs nothing from the registry. A flavour is built on the base built from the
same tree, not on the version it pins, and every other layer comes from a public image.

A pull request runs code its author wrote, so a workflow it triggers holds a read-only
token and nothing else: no secret, no registry login, no route to the registry, nothing that
pushes an image. `test-workflows` enforces that on every `verify`: it reads each workflow's
`on:` block and, for any with a `pull_request` or `pull_request_target` trigger, fails on a
line that names a secret, the publish route, a registry login or an image push. It reads
text, so a comment that names one fails as well, and a workflow whose triggers it cannot
find is refused. The checker is itself tried against fixture workflows on every run.

## Publishing

Images are published to the private Harbor `devcontainer-images` project as immutable
SemVer tags (`MAJOR.MINOR.PATCH`) with an SBOM attestation. Each image has its own
version line, so releasing one image publishes nothing on any other. A moving `stable`
tag on each image points at its most recent version that passed the verification ladder.
Consumers pin a version tag, which Renovate keeps current. Pushes authenticate as the
project-scoped robot account only, never an admin user.

The bump level says what a release changes for a consumer, so a pin can take a minor or a
patch without thought and a major with a look:

- **major:** something a consumer may rely on is gone or behaves differently: a tool or an
  SDK removed, a new base OS release, a changed user or path.
- **minor:** something is added (a tool, an SDK) and nothing is removed.
- **patch:** the same contents at newer patch levels.

A release takes the image and the level and works out the number: the next one after the
highest version that image already holds in the registry. Git marks it with the tag
`<image>-<version>` (`dotnet-1.4.0`). An image whose own directory is unchanged since its
last tag has nothing to release, so a change to CI, tasks or docs publishes nothing.
Rebuilding unchanged contents on purpose, to pick up upstream patches, takes the refresh
option.

A flavour names the base version it stands on in its own Dockerfile, and a release builds
on exactly that version. A new base therefore reaches a flavour the way it reaches any
other consumer: Renovate raises the pin, and the flavour's next release ships it. Every
published image records the commit it was built from and the image it stands on in its
labels.

Harbor is fronted by a Cloudflare tunnel whose HTTP hostname caps a request body far below
an image layer, so a push from outside the LAN cannot use that hostname. There are two
publishers:

- **CI, `build.yaml`:** run by hand on the default branch with the image (or `changed`,
  for every image whose directory changed since its last release), the bump level and
  the refresh option. Images release side by side on GitHub-hosted runners, each with a
  GitHub release and generated notes, and one release of an image runs at a time. The
  runner reaches Harbor through the tunnel's TCP route, opened by the `harbor-route`
  action, which admits only an Access service token.
- **Rung 3, workstation:** `publish <image> <major|minor|patch>` is the same release from
  inside the LAN, for when CI or the route is unavailable. It runs the image's rungs of
  the ladder, builds from a committed tree with `buildx` on a `docker-container` builder
  (the default driver cannot push attestations), then pushes the version and its release
  tag.
  `promote-stable <image> <version>` re-tags a published version as `stable` by digest,
  so nothing is rebuilt and the attestations ride along. Both tasks read
  `HARBOR_ROBOT_USERNAME` / `HARBOR_ROBOT_SECRET` from the environment (the secret is the
  Key Vault entry `harbor-gha-robot-creds`) and refuse any non-robot username; `publish`
  also refuses to run if the registry does not resolve to a private address, and logs in
  under a throwaway `DOCKER_CONFIG` so the workstation's own registry login is left alone.

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
