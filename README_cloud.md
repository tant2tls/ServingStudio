# ServingStudio on a cloud GPU container

Setup for a single-user rented GPU box: Ubuntu 24.04, logged in as `root`, Docker
daemon already running inside the container. Verified on 2026-10-02 on one
H100 80GB (driver 580, CUDA 13.0). See [README.md](README.md) for the product
overview and [reproduce.md](reproduce.md) for the full reference.

Allow about 20 minutes and 45 GB of disk (25 GB of it is the Agent runner image).

## 1. Pick a workspace directory Docker can mount

Agent runners bind-mount the workspace, so it must sit on a filesystem Docker can
mount. Encrypted/FUSE home volumes cannot be mounted (here `/root` is `gocryptfs`),
so use `/workspace`. Test your chosen parent directory first:

```bash
mkdir -p /workspace
docker run --rm -v /workspace:/mnt ubuntu:24.04 true && echo "mount ok"

# The image presets HF_HOME; the Agent mounts it into runners, so it must exist.
[ -n "$HF_HOME" ] && mkdir -p "$HF_HOME"
```

## 2. Install host tools

```bash
apt-get update
apt-get install -y build-essential ca-certificates cmake curl git mold \
  ninja-build pkg-config protobuf-compiler python3 tmux xz-utils

# Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs

# Rust, just, uv (skip uv if `uv --version` already works)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to /usr/local/bin
curl -LsSf https://astral.sh/uv/install.sh | sh

source ~/.cargo/env   # repeat in every shell that was open before this step
```

## 3. Clone and build

Run this inside `tmux`; the build takes about 6 minutes.

```bash
cd /workspace
git clone --recurse-submodules \
  https://github.com/tant2tls/ServingStudio.git servingstudio
cd servingstudio

just setup-env
just check-tools
just check-gpu      # nvidia-smi from inside a Docker container

# Cargo.lock is not tracked in ServingStudioSim, and the build uses --locked.
(source .env && cd ServingStudioSim && uv run --frozen cargo generate-lockfile)

just build
```

## 4. Configure the Agent provider

This must exist before the runner image is built. Minimal Claude setup using the
host login in `~/.claude/.credentials.json` (run `claude` once and log in first):

```bash
cat > ServingStudioAgent/providers.yaml <<'EOF'
version: 1

providers:
  claude:
    adapter: claude
    label: Claude
    home: ~/.claude
    default_model: claude-sonnet-5
    default_effort: high
    models:
      claude-sonnet-5:
        efforts: [low, medium, high, xhigh, max]
      claude-opus-5-5:
        efforts: [low, medium, high, xhigh, max]

defaults:
  orchestrator: claude
  implementer: claude
  assistant: claude
EOF
chmod 600 ServingStudioAgent/providers.yaml
```

For Codex, API tokens, or gateways, start from
`ServingStudioAgent/examples/providers.yaml` and see
[providers.md](ServingStudioAgent/doc/providers.md).

## 5. Build the runner image

About 10 minutes, inside `tmux`:

```bash
just build-runner-image
docker image inspect vibesim-agent-runner:root >/dev/null && echo "image ok"
```

> [!NOTE]
> The command currently ends with
> `runner image smoke failed: Cargo target seed caused a third-party dependency to recompile`
> (triggered by the first-party `req-frontend` crate). The image is already built
> and tagged by then; continue if `image ok` prints.

## 6. Start

```bash
just agent-init     # once per new workspace
just start
just smoke-local    # expect six lines of "200"
```

As `root` the ports are UI `60030`, Agent `60031`, Analyzer `60032`. Forward the UI
port from your laptop and open <http://localhost:60030>:

```bash
ssh -L 60030:localhost:60030 <user>@<host> -p <ssh-port>
```

## Change ports

The services use three consecutive ports from `VIBESIM_PORT_BASE` (1024–65533):
UI = base, Agent = base + 1, Analyzer = base + 2. Only the UI port needs to be
reachable from outside. Example for UI on `8080`:

```bash
echo 'export VIBESIM_PORT_BASE=8080' >> .env
just stop
just check-ports    # all three must be free; Jupyter often holds 8888
just start
```

Then forward the new UI port instead: `ssh -L 8080:localhost:8080 ...`.

## Daily commands

| Command | Purpose |
| --- | --- |
| `just services-status` | List the running service sessions. |
| `just restart` | Restart after a reboot, pull, or config change. |
| `just stop` | Stop this workspace's services. |
| `tail -f tmp/{backend,analyzer,ui}.log` | Service logs. |

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `missing: cargo` | `source ~/.cargo/env`, then restart the tmux server (`tmux kill-server`) so new sessions inherit the `PATH`. |
| `cannot create the lock file ... Cargo.lock` | Run the `cargo generate-lockfile` line from step 3. |
| `Provider configuration file is required` | Do step 4 before step 5. |
| `error mounting "..." to rootfs` | The workspace is on a filesystem Docker cannot mount; redo step 1 and clone elsewhere. |
| UI shows `Agent unavailable`; `tmp/backend.log` has `HF_HOME directory must exist` | `mkdir -p "$HF_HOME"` and retry; no restart needed. |
| `Port check failed` | Pick another base; see [Change ports](#change-ports). |
