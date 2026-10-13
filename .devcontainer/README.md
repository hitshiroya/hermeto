# Hermeto Dev Container

## What is a dev container?

A dev container is a fully configured development environment running inside
a Docker or Podman container. Instead of each contributor installing Python,
Go, Node.js, Cargo, Ruby, and the right versions of every tool locally, the
container ships everything pre-installed and identically configured for
everyone.

The key idea: your **source code stays on your machine** (bind-mounted into
the container), while the **tools and runtime live inside the container**. You
edit files in your normal editor — the container sees the changes instantly.

Dev containers are defined by two files in `.devcontainer/`:

- `Dockerfile` — what goes into the image (tools, runtimes, dependencies)
- `devcontainer.json` — how the container runs (mounts, env vars, startup
  commands)

They are consumed by the `devcontainer` CLI (or VS Code, but VS Code is not
required).

### Why use one for Hermeto?

Hermeto's test suite requires Go, Node.js, Cargo, Ruby, and Python — all at
specific versions, all on the same machine. Setting this up manually is
error-prone and differs across macOS, Fedora, and Ubuntu. A dev container
makes the environment a checked-in artifact: reproducible, shareable, and
disposable.

---

A CLI-first dev container for Hermeto. No VS Code required — works with
any editor via the `devcontainer` CLI.

## How it works

The dev image extends the published Hermeto production image
(`ghcr.io/hermetoproject/hermeto:latest`) rather than rebuilding from
scratch. This means Go, Node.js, Cargo, Ruby, and all Hermeto runtime
dependencies are already present. Only three things are layered on top:

- `python3.12-pip` — stripped by the production multi-stage build
- A `/devenv/` venv with all dev/test tools pre-installed (`nox`, `ruff`,
  `mypy`, `pytest`, etc.)
- `createrepo_c` copied from the production `/venv/` — already compiled,
  no C compiler needed

At container start, a single `postCreateCommand` runs:

```
/devenv/bin/pip install -e /workspace
```

This installs Hermeto from your bind-mounted source in editable mode.
Change a `.py` file in your editor — the running CLI picks it up
immediately, no reinstall needed.

## Prerequisites

1. **devcontainer CLI**

   ```sh
   npm install -g @devcontainers/cli
   ```

2. **A container runtime** — Docker Desktop or Podman

   If using Podman, point the devcontainer CLI at its socket:

   ```sh
   export DOCKER_HOST="unix://$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}')"
   ```

   Add that line to `~/.zshrc` (or `~/.bashrc`) to make it permanent.

## Quick start

```sh
# 1. Start the container (builds image on first run, ~45s)
devcontainer up --workspace-folder .

# 2. Open a shell inside
devcontainer exec --workspace-folder . bash

# 3. You land in /workspace — your source, all tools on PATH
hermeto --help
nox -s lint
nox -s python
```

## Day-to-day workflow

| Task | Command (inside container) |
|---|---|
| Run linters | `nox -s lint` |
| Run unit tests | `nox -s python` |
| Run a single test file | `nox -s python -- tests/unit/test_cli.py` |
| Use the Hermeto CLI | `hermeto fetch-deps ...` |
| Exit shell | `exit` (container keeps running) |

## Returning after closing the laptop

The container persists across sessions. On next login:

```sh
# if using Podman and DOCKER_HOST is not in .zshrc
export DOCKER_HOST="unix://$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}')"

# restart the existing container (~2s, no rebuild)
devcontainer up --workspace-folder .

devcontainer exec --workspace-folder . bash
```

## Rebuilding the image

A rebuild is only needed when `.devcontainer/Dockerfile` or
`requirements-extras.txt` changes:

```sh
devcontainer up --workspace-folder . --rebuild
```

## What dev containers give you

| Benefit | Details |
|---|---|
| Reproducible environment | Every contributor runs the same OS, Python, Go, Node, Cargo, Ruby — no "works on my machine" |
| Zero local install | No Python venv, Go, Node, or Cargo needed on the host. One `devcontainer up`. |
| Live edits | Source is bind-mounted. Edit in your editor on the host → change is instantly visible inside the container |
| Fast restart | `devcontainer up` detects the existing container and restarts it in ~2s. Rebuilds only when `Dockerfile` changes. |
| Hermeto image as base | All runtimes and compiled deps come from the production image — nothing duplicated or re-downloaded |
| Editable install | `hermeto` is `pip install -e`'d from `/workspace` — change a `.py` file and the CLI picks it up immediately |
| CLI-first | No VS Code required. Works with any editor or terminal. |

## Personal configuration — layering your tools

### Option A — Dotfiles repository (recommended)

Maintain a personal GitHub repo with your dotfiles (`.vimrc`, `.zshrc`, git
aliases, debugger configs, etc.). The devcontainer CLI clones it into the
container and runs your `install.sh` automatically at startup.

```sh
devcontainer up --workspace-folder . \
  --dotfiles-repository https://github.com/you/dotfiles
```

Your `install.sh` symlinks configs into `$HOME`. Personal, versioned, zero
shared-config pollution. The shared `devcontainer.json` stays untouched.

### Option B — Local override file (gitignored)

Create `.devcontainer/devcontainer.local.json` (already gitignored) for
machine-specific overrides — extra mounts, env vars, or devcontainer Features:

```json
{
  "mounts": [
    "source=${localEnv:HOME}/.gitconfig,target=/root/.gitconfig,type=bind,readonly",
    "source=${localEnv:HOME}/.ssh,target=/root/.ssh,type=bind,readonly"
  ],
  "containerEnv": {
    "MY_API_KEY": "${localEnv:MY_API_KEY}"
  }
}
```

Use it with:

```sh
devcontainer up --workspace-folder . \
  --override-config .devcontainer/devcontainer.local.json
```

> **Important**: `--override-config` replaces the base `devcontainer.json`
> entirely — it does not merge. Your local file must repeat all required fields
> (`build`, `workspaceFolder`, `workspaceMount`, `containerEnv`) plus your
> additions. Copy the base config as a starting point and add your overrides.

### What persists

| Mechanism | Survives restart? | Survives rebuild? | Requires rebuild? |
|---|---|---|---|
| Dotfiles repo | Re-runs `install.sh` (fast) | Re-runs `install.sh` (fast) | Never |
| Local override mounts | Yes — bind mount | Yes — bind mount | Never |
| Tools installed ad-hoc inside container | **No** — lost on recreate | No | — |


## Limitations

### This setup

| Limitation | Notes |
|---|---|
| Container runtime required | Docker Desktop or Podman must be running before `devcontainer up` |
| No Podman auto-detection | devcontainer CLI defaults to Docker. Podman users must set `DOCKER_HOST` (see Prerequisites) |
| Rebuild on dependency change | If `requirements-extras.txt` or `Dockerfile` changes, run `devcontainer up --rebuild` |
| `createrepo_c` version coupling | Compiled `.so` copied from prod image — must rebuild dev image if prod upgrades it |
| Runs as root | UBI10 default. Fine for local dev, not for shared environments |

### Dev containers in general

**Slow bind mount I/O on macOS**
On macOS, Docker and Podman run inside a Linux VM. Every file read/write from
the container crosses that VM boundary. For light editing this is invisible,
but for workloads that touch many files rapidly — running a large test suite,
compiling, or `pip install` with many packages — it is noticeably slower than
native Linux. Mitigation: use named Docker volumes for write-heavy paths
(e.g. build caches) instead of bind mounts.

**Large image sizes**
Once you layer a runtime base plus dev tools, images easily reach 2–5 GB.
Running multiple projects or keeping old image versions accumulates disk fast.
Clean up regularly: `docker system prune -a` removes unused images and
build cache.

**Cold start on first build**
The first `devcontainer up` downloads the base image and runs every
`RUN` step in the Dockerfile. Depending on what is installed this can take
anywhere from 1 to 10 minutes. Subsequent starts reuse the cached image and
take seconds. Sharing a pre-built image via a registry eliminates this for
the whole team.

**No GUI applications**
Dev containers have no display server. You cannot run a browser, a GUI
debugger, or any desktop application inside. Text-based tools (vim, tmux,
gdb CLI) work fine. If you need a GUI, it must run on the host and connect
to a service exposed by the container via port forwarding.

**Docker-in-Docker**
Running containers inside the dev container — for example to spin up test
dependencies with `podman-compose` — requires either privileged mode or
mounting the host's container socket. Both approaches have security
implications. 

**Resource consumption**
The container runtime VM (especially on macOS) runs continuously and
consumes roughly 2–4 GB of RAM regardless of what is happening inside.
On machines with 8 GB RAM this competes noticeably with the IDE and browser.
Podman's `libkrun` backend tends to use less memory than Docker Desktop's
full Linux VM.

## Files

| File | Purpose |
|---|---|
| `Dockerfile` | Dev image definition — extends the Hermeto production image |
| `devcontainer.json` | devcontainer spec — mounts, env, startup command |
