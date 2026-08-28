# Agent Sandbox (`asb`)

A minimal, hardened Docker environment for running coding agents without giving
them your user account. The agent runs as a non-root user inside a container
whose **only writable window onto the host is a single bind-mounted directory**.
Files it creates there are owned by *you* on the host, because the container runs
as your uid/gid.

## Quick start

```sh
./asb build          # build the image with your uid/gid baked in
mkdir -p home        # the host dir that becomes the container's HOME
./asb start          # launch (detached)
./asb sh             # drop into a shell in the container; run your agent here
./asb stop           # tear down (the container is --rm, so nothing persists)
```

Put `asb` on your `PATH` (or symlink it) to use it from anywhere.

## Commands

| Command | Description |
|---|---|
| `asb build [args]`   | Build the image (passes your uid/gid as build args) |
| `asb rebuild [args]` | Build with `--no-cache` |
| `asb start`          | Launch the container (detached, `--rm`) |
| `asb stop`           | Stop the container |
| `asb restart`        | Stop then start |
| `asb sh [cmd...]`    | Exec into the running container (default: a shell) |
| `asb run <cmd...>`   | One-shot: throwaway container, run cmd, remove |
| `asb status`         | Show container state |
| `asb logs [args]`    | Container logs |
| `asb config`         | Print the resolved configuration |
| `asb help`           | Help |

Global flags: `--dry-run`/`-n` (print the `docker` command instead of running
it — inspect before you trust it), `--name <n>`, `--image <t>`.

## Configuration

Precedence: **command-line flags > environment (`ASB_*`) > `asb.conf` > defaults.**

Copy `asb.conf.example` to `asb.conf` and edit, or export `ASB_*` variables.
Run `asb config` to see the effective values. Key ones:

| Variable | Default | Meaning |
|---|---|---|
| `ASB_UID` / `ASB_GID` | your `id -u` / `id -g` | uid/gid the container runs as → host file ownership |
| `ASB_USER`      | `agent`            | in-container username / HOME (`/home/<user>`) |
| `ASB_HOME`      | `<asb dir>/home`   | host dir bind-mounted as HOME (the writable surface) |
| `ASB_WORKSPACE` | *(none)*           | optional second host mount |
| `ASB_PORTS`     | *(none)*           | published ports, comma list, e.g. `5173-5177,9323` - a bare port/range keeps the same number on the host; use `host:container` to remap |
| `ASB_NETWORK`   | `bridge`           | `bridge` \| `none` \| custom network |
| `ASB_CPUS` / `ASB_MEMORY` / `ASB_PIDS_LIMIT` | `4` / `16g` / `2048` | resource caps |
| `ASB_READONLY` | `0` | `1` = read-only root filesystem |
| `ASB_TMPFS_EXEC` | `0` | `1` = allow exec on `/tmp` (default is `noexec`) |

## Security model

The isolation boundary is deliberately simple: **the agent can read/write only
what you bind-mount, as your uid, with a reduced kernel surface.**

What `asb start` enforces by default:

- **Non-root, your uid/gid** — `--user $ASB_UID:$ASB_GID`; no privilege to touch
  host files outside the mount, and created files are owned by you.
- **`--security-opt no-new-privileges`** — setuid binaries can't escalate.
- **`--cap-drop ALL`** — drops every Linux capability.
- **`--pids-limit`** — caps fork bombs.
- **`--cpus` / `--memory`** — resource ceilings.
- **`/tmp` = tmpfs, `noexec,nosuid`** — no executing dropped payloads from `/tmp`.
- **`--rm`** — the container is disposable; state lives only in the bind mount.
- Optional **`--read-only`** rootfs and **`--network none`** for tighter runs.

What it does **not** do: this is a container, not a VM. A kernel exploit escapes
it. Don't run it privileged, don't add `--cap-add` you don't need, and keep the
bind mount scoped to exactly the directory the agent should touch. For network
egress control (blocking exfiltration), put the container on a custom network
with an egress firewall/proxy, or use `ASB_NETWORK=none` when the agent doesn't
need the internet.

Inspect any command before running it: `asb start --dry-run`.
