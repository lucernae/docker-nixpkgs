# CLAUDE.md — docker-nixpkgs

This file documents the structure, conventions, and development workflows for the `docker-nixpkgs` repository. It is intended to help AI assistants and new contributors understand the codebase quickly.

## Project Overview

`docker-nixpkgs` is an automated collection of Docker images built using [Nix](https://nixos.org/) and the latest [nixpkgs](https://github.com/NixOS/nixpkgs) package set. All images are refreshed daily with the latest versions from nixpkgs.

- **Maintained by**: [nix-community](https://github.com/nix-community)
- **License**: MIT
- **Registries**: Docker Hub (`nixpkgs/<image>`), GHCR (`ghcr.io/nix-community/docker-nixpkgs/<image>`), and `docker.nix-community.org`
- **Architectures**: `x86_64-linux`, `aarch64-linux`

## Repository Structure

```
docker-nixpkgs/
├── images/                     # One subdirectory per Docker image
│   ├── bash/default.nix
│   ├── busybox/default.nix
│   ├── cachix/default.nix
│   ├── cachix-flakes/default.nix
│   ├── caddy/default.nix
│   ├── curl/default.nix
│   ├── devcontainer/default.nix
│   ├── devenv/default.nix
│   ├── docker-compose/default.nix
│   ├── hugo/default.nix
│   ├── kubectl/default.nix
│   ├── kubernetes-helm/default.nix
│   ├── nginx/default.nix
│   ├── nix/default.nix
│   ├── nix-flakes/default.nix
│   ├── nix-unstable/default.nix
│   └── nix-unstable-static/default.nix
├── lib/
│   ├── buildCLIImage.nix       # Reusable helper for simple CLI images
│   ├── importDir.nix           # Auto-discovers image subdirectories
│   └── mkUserEnvironment.nix   # Builds nix-env compatible environments
├── .github/
│   ├── workflows/
│   │   ├── nix.yml             # Main CI: build + push (nix-shell based)
│   │   └── nix-build-flake.yml # Alternate CI: per-image flake builds
│   └── dependabot.yml
├── flake.nix                   # Nix Flakes entry point (multi-channel)
├── flake.lock                  # Pinned flake inputs
├── overlay.nix                 # Nix overlay extending nixpkgs
├── default.nix                 # Non-flakes compatibility entry point
├── pkgs.nix                    # Package set definition (non-flakes)
├── shell.nix                   # Development shell (dive, jq, skopeo, podman)
├── ci.sh                       # CI build + push orchestrator
├── ci-manifests.sh             # CI multi-arch manifest creation
├── docker-login                # Authenticate with a container registry
├── dockerhub-metadata          # Update Docker Hub descriptions
├── generate-manifests          # Create multi-arch manifests via podman
├── push-all                    # Push all images to a registry via skopeo
├── readme-image-matrix         # Generate README image table (mdsh)
├── .gitlab-ci.yml              # GitLab CI configuration
├── .envrc                      # direnv integration
└── README.md
```

## Channels and Image Tags

Each image is built from multiple nixpkgs channels. The channel determines the package versions inside the image.

| Channel         | Docker Image Tag | Description                              |
|-----------------|------------------|------------------------------------------|
| `nixos-unstable` | `latest`        | Latest/greatest; major versions may change |
| `nixos-24.05`   | `nixos-24.05`    | Stable; security updates only            |
| `nixos-24.11`   | `nixos-24.11`    | Stable; security updates only            |
| `nixos-25.05`   | `nixos-25.05`    | Stable; security updates only            |
| `nixos-25.11`   | `nixos-25.11`    | Stable; security updates only            |

`nixos-unstable` always maps to the `latest` Docker tag. All other channels map to their channel name as the tag.

## Key Conventions

### Nix File Conventions

- **Overlay pattern**: `overlay.nix` uses the `_: pkgs:` function signature.
- **Image discovery**: `lib/importDir.nix` automatically scans `./images/` and imports every subdirectory's `default.nix`.
- **Image definition**: Every image lives at `./images/<image-name>/default.nix` and receives `pkgs` attributes via `callPackage`.
- **Image names**: Must be lowercase (Docker requirement).
- **Environment variable for channel**: `NIXPKGS_CHANNEL` is read by `overlay.nix` (`flakeParameters.nixpkgsChannel`) and by CI scripts.

### Adding a New Image

1. Create `./images/<image-name>/default.nix`.
2. For simple single-binary images, use `buildCLIImage`:

   ```nix
   { buildCLIImage, <package> }:
   buildCLIImage {
     drv = <package>;
   }
   ```

3. For complex images, use `pkgs.dockerTools.buildLayeredImage` or `buildImageWithNixDb` directly.
4. Test locally: `nix-build -A <image-name>` then `docker load -i result`.

### Adding a New Channel

When a new nixpkgs channel is released (e.g., `nixos-25.11`), update **all** of the following locations:

| File | What to change |
|------|---------------|
| `flake.nix` | Add input `nixpkgs-25-11`, add `pkgs-25-11` instantiation, add `"nixos-25.11"` output |
| `.github/workflows/nix.yml` | Add `nixos-25.11` to `jobs.build.strategy.matrix.channel` and `jobs.push-manifest.strategy.matrix.channel` |
| `.github/workflows/nix-build-flake.yml` | Add `nixos-25.11` to `jobs.build.strategy.matrix.nix_channel` and `jobs.manifests-create.strategy.matrix.nix_channel` |
| `.gitlab-ci.yml` | Add `nixos-25.11` under `NIXPKGS_CHANNEL` in the build matrix |
| `README.md` | Add a row to the Channels table |

### Image Complexity Levels

1. **Simple CLI images** (`curl`, `bash`, `kubectl`, etc.): Use `buildCLIImage` helper (~5 lines of Nix).
2. **Extension images** (`cachix-flakes`, `devenv`): Override another image with `extraContents`.
3. **Full environment images** (`nix`, `devcontainer`): Use `dockerTools.buildImageWithNixDb` or `buildImage` with a full root filesystem under `./images/<name>/root/`.

### Standard Image Contents

All images built with `buildCLIImage` include:
- `busybox` — provides `/bin/sh`
- `cacert` — CA certificates for TLS; `SSL_CERT_FILE` is set automatically

### Image Tagging Format (Registry Push)

```
<registry>/<prefix>/<image>:<tag>-<system>
```

Examples:
- `docker.io/nixpkgs/curl:latest-x86_64-linux`
- `ghcr.io/nix-community/docker-nixpkgs/curl:nixos-25.05-aarch64-linux`

Multi-arch manifests are created by `generate-manifests` / `ci-manifests.sh` after per-architecture images are pushed.

## Development Workflow

### Local Development Shell

```bash
nix-shell        # or: nix develop
# Provides: dive, jq, skopeo, podman
```

The `shell.nix` uses `nixos-25.05` as its base channel.
The `.envrc` activates the shell automatically when using direnv.

### Building Images Locally

```bash
# Build all images (non-flakes)
nix-build --no-out-link --option sandbox true --argstr system x86_64-linux

# Build a specific image
nix-build -A <image-name>

# Build via Flakes (specific image + channel + system)
nix build '.#docker-nixpkgs.x86_64-linux."nixos-unstable".<image-name>'

# Load and test
docker load -i $(nix-build -A <image-name> --no-out-link)
docker run --rm nixpkgs/<image-name>
```

### Environment Variables Used by CI Scripts

| Variable           | Default            | Purpose                                        |
|--------------------|--------------------|------------------------------------------------|
| `NIXPKGS_CHANNEL`  | `nixos-unstable`   | Which nixpkgs channel to use                   |
| `CI_REGISTRY`      | `docker.io`        | Container registry to push to                  |
| `CI_REGISTRY_AUTH` | _(empty)_          | `user:token` for registry authentication       |
| `CI_PROJECT_PATH`  | `nixpkgs`          | Image name prefix in the registry              |
| `NIX_SYSTEM_NAME`  | `x86_64-linux`     | Target architecture                            |

### ci.sh Flow

1. Build all images: `nix-build --no-out-link --option sandbox true --argstr system $NIX_SYSTEM_NAME`
2. Exit if not on `master` branch.
3. Login to registry via `./docker-login` if `CI_REGISTRY_AUTH` is set.
4. Push all images via `./push-all`.
5. Update Docker Hub metadata if pushing to `docker.io`.

### Multi-Architecture Manifests

After both `x86_64-linux` and `aarch64-linux` builds complete, `ci-manifests.sh` calls `./generate-manifests` to combine them into a single multi-arch manifest using `podman manifest`.

## CI/CD

### GitHub Actions (`nix.yml`) — Primary Pipeline

- **Triggers**: push to `master`, pull requests, `workflow_dispatch`, daily cron at 00:00 UTC.
- **Matrix**: `channel × system` (5 channels × 2 systems = 10 build jobs).
- Pushes to both Docker Hub and GHCR.
- `push-manifest` job runs after all builds and creates multi-arch manifests.

### GitHub Actions (`nix-build-flake.yml`) — Flakes Pipeline

- Builds each image individually using `nix build '.#docker-nixpkgs...'`.
- Matrix: `nix_channel × nix_system × target_image`.
- Pushes to GHCR only.
- Used for testing the Flakes output structure.

### GitLab CI (`.gitlab-ci.yml`)

- Legacy support; minimal single `build` stage.
- Uses `nixpkgs/nix:nixos-23.05` as the CI image.
- Same channel matrix, no architecture matrix.

## Nix Architecture

### flake.nix

The flake has one input per nixpkgs channel (`nixpkgs`, `nixpkgs-24-05`, `nixpkgs-24-11`, `nixpkgs-25-05`, `nixpkgs-25-11`). For each system, it produces:

```
docker-nixpkgs.<system>."<channel>".<image-name>
```

Each channel package set is instantiated with `overlay.nix` applied and `flakeParameters.nixpkgsChannel` set to the channel string.

### overlay.nix

Extends nixpkgs with:
- `buildCLIImage` — reusable Docker image helper
- `flakeParameters` — runtime channel injection (reads `NIXPKGS_CHANNEL` env var)
- `docker-nixpkgs` — attribute set of all image derivations (auto-discovered via `importDir`)
- `mkUserEnvironment` — builds nix-env compatible environments
- `gitReallyMinimal` — Git without Perl/Python/manual pages

### lib/buildCLIImage.nix

Template for simple single-binary Docker images:
- Uses `dockerTools.buildLayeredImage` for efficient layer reuse.
- Adds `busybox` and `cacert` to every image.
- Sets `SSL_CERT_FILE` and `PATH`.
- Default `Cmd` = `["/bin/<binName>"]`.

### lib/importDir.nix

Reads a directory, filters to subdirectory entries, and maps each to an import call. Used to auto-discover all image definitions without manual registration.

## Useful Commands

```bash
# Format Nix code
nix fmt

# Check what's in an image layer
dive $(docker images -q nixpkgs/curl:latest)

# Copy an image between registries
skopeo copy docker://nixpkgs/curl:latest oci:/tmp/curl

# Inspect a derivation
nix show-derivation '.#docker-nixpkgs.x86_64-linux."nixos-unstable".curl'

# List all image attributes
nix eval '.#docker-nixpkgs.x86_64-linux."nixos-unstable"' --apply builtins.attrNames
```
