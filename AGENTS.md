# infra-toolkit

## What it is

Utility container image forked from `strangelove-ventures/infra-toolkit`. It bundles a curated set of static binaries and Alpine packages into a minimal image used for init and maintenance tasks in Cosmos node deployments. It is not a standalone application.

## Where it's used

The BryanLabs fork of `cosmos-operator` defaults its init containers to `ghcr.io/bryanlabs/infra-toolkit` instead of the upstream strangelove image. Tasks include file permission fixes (chmod/chown), genesis/addrbook fetching, config merging, and node data preparation. It runs as an init container or sidecar, not as a Deployment.

## BryanLabs customizations

All commits in this repo are the full history (the fork diverged at the initial commit). BryanLabs additions beyond the upstream baseline:

- `aria2c` added for parallel/multi-source downloads (PR #19)
- `busybox.min.config` expanded to enable `rm`, `mv`, `ln`, `vi`, `mkdir`, `sed` (PRs #20-23)
- `procps` added to Alpine layer (PR #16)
- `gnu tar` from Alpine and `rsync` added (PRs #12, #14)
- `zstd-dev` added for zstandard compression support (PR #11)
- `nc` and `ping` enabled in busybox config (PR #9)
- Main `Dockerfile` pins `config-merge` to `0.2.1` (vs `latest` in `native.Dockerfile`)
- Main `Dockerfile` adds cross-compilation support (`BUILDPLATFORM`/`TARGETARCH` args, musl cross toolchains)

## How it works

Multi-stage build produces a single Alpine-based image with:

- **busybox v1.34.1** (static, custom config): ash shell, tar, sed, vi, grep, ls, ln, rm, mv, mkdir, nc, ping, nslookup, less, watch, tee, cat, head, tail, du, df, date, env, sleep, and others
- **jq 1.6** (static): JSON processing
- **config-merge** (from `boxboat/config-merge:0.2.1`): TOML/YAML/JSON config merging with `envsubst`
- **dasel v1.26.0** (pinned binary with sha256 check): structured data query/update (TOML, YAML, JSON, CSV, XML)
- **Alpine packages**: curl, wget, lz4, zstd-dev, rsync, tar (GNU), nano, npm, procps, aria2

There is no custom entrypoint or CMD. The cosmos-operator supplies the command for each init container.

## Code map

```
Dockerfile           # Primary multi-arch build (amd64 + arm64 cross-compile support)
native.Dockerfile    # Single-arch variant (no cross-compile, config-merge:latest)
node.Dockerfile      # Experimental: fully-static Node.js 18 from source (scratch image)
busybox.min.config   # Busybox .config selecting enabled applets
.github/workflows/
  docker-publish.yaml  # CI: builds and pushes on any branch/tag push
```

## Build and deploy

**Image:** `ghcr.io/bryanlabs/infra-toolkit`

CI builds multi-arch (`linux/amd64,linux/arm64`) via GitHub Actions on every push. Tags follow branch name; `latest` tracks `main`.

Local build:
```sh
docker buildx build --builder worker1 --platform linux/amd64 -t ghcr.io/bryanlabs/infra-toolkit:dev .
```

## Gotchas

- `node.Dockerfile` builds a fully static Node.js binary from source into a `scratch` image. It is not published by CI (only `Dockerfile` is referenced in the workflow). It exists as a standalone utility and is not part of the normal image.
- Busybox is built statically from source (v1.34.1). Changes to enabled applets require editing `busybox.min.config` and rebuilding.
- `dasel` binaries are pinned with hardcoded sha256 checksums because the upstream project does not publish checksums. Update both the binary URL and checksum together if upgrading dasel.
- The upstream (`strangelove-ventures/infra-toolkit`) remote is tracked locally. Keep it to pull upstream fixes when needed.
