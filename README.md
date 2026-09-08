# devcontainer-images

Golden dev-environment images for workstation devpods: a `base` image carrying the
common tool belt, and thin language flavors layered on top (`dotnet`). Images are
deterministic Docker builds — base layers from public registries, tool versions pinned
via `base/mise.toml` + `base/mise.lock` — published to the Harbor `devcontainer-images`
project and consumed by repos through `devcontainer.json`. Personal config never bakes
in; it lands at devpod start-up via the dotfiles hook.

## Layout

- `base/` — base Dockerfile + mise config/lockfile (the pinned tool belt)
- `dotnet/` — .NET flavor, `FROM` base
- `tasks/` — build/test entry points, run via `mise run <task>` (e.g. `build-base`,
  `test-base`, `build-dotnet`)
- `tests/` — image verification
- `renovate.json` — automated version-bump PRs for the pins

Bumping one tool re-runs only the `mise install` layer; OS layers stay cached, so a
version bump ships as one small content-addressed blob.

## Prune safety

The local Docker image cache is the standing fallback when Harbor is unreachable: the
last-known-good `devcontainer-base` and `devcontainer-dotnet` images let a devpod start
with no registry at all. A blanket `docker system prune -a` (or `docker image prune -a`)
evicts every image not currently used by a container — including those fallbacks — so
`-a` prunes are the one habit that silently deletes the safety net.

When reclaiming space:

- `docker image prune` (no `-a`) — removes dangling build residue only; tagged flavor
  images are never touched. This is the safe default.
- `docker system prune` (no `-a`) — same rule: stopped containers, networks and dangling
  images, but no tagged images.
- If a deeper clean is genuinely needed, re-pull or rebuild the flavors first
  (`mise run build-base`, `mise run build-dotnet`) so a last-known-good image exists
  before anything is evicted.
