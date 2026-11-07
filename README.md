# safe-code

`safe-code` provides command-line launchers for running AI coding tools with
limited access to the host filesystem, using
[Bubblewrap](https://github.com/containers/bubblewrap) on Linux.

The first committed launcher is **safe-opencode**, for OpenCode.
Future launchers may support tools such as **Claude Code** and **Kimi Code**
using a similar approach. These additional tools are not supported yet.

This may be useful on shared or managed cluster systems where Docker is disabled
or unavailable, and where users have few other practical isolation tools. It
does not require a Docker daemon, but it does require Bubblewrap to be installed
and permitted by the system.

## Security scope

This project has **not undergone an extensive security audit**. Its settings
provide some hardening compared with running an AI coding tool without
filesystem isolation, but this should not be treated as a security boundary or
as protection for highly sensitive data.

The launcher limits which host paths are visible to the tool and can make the
project read-only by default. However, network access remains available in the
host network namespace, and the level of isolation depends on the host kernel,
Bubblewrap configuration, and local system policy. A writable mount gives the
tool permission to change files in that mounted directory.

Use only on systems and with data for which you are authorized. Do not mount
credentials, broad home directories, or other sensitive paths. On clusters,
follow local policy and ask administrators whether Bubblewrap and user
namespaces are supported.

## Common requirements

- Linux.
- Bubblewrap (`bwrap`), installed and usable by the current account. The host
  must allow user namespaces or provide an appropriately configured Bubblewrap
  installation.
- `bash`, `realpath`, and `readlink`.

## `safe-opencode`

The first launcher targets OpenCode V2. It requires OpenCode available as
`opencode` in the host `PATH`, or a native binary selected with
`SAFE_OPENCODE_BIN`.

The script creates host OpenCode config and data directories if they do not
exist. Defaults are `${XDG_CONFIG_HOME:-$HOME/.config}/opencode` and
`${XDG_DATA_HOME:-$HOME/.local/share}/opencode`.

### Installation and minimal setup

Choose either installation method below. Bubblewrap must be installed separately
through your system's package manager or administrator. **For filesystem
isolation, the OpenCode server must be launched within `safe-opencode`, not as
a background service on the host.** Both setups disable automatic
background-service startup and stop any existing service.

#### Option 1: curl (no Node.js required)

The installer downloads a self-contained native OpenCode V2 binary:

```bash
curl -fsSL https://opencode.ai/v2/install | bash
# Reopen your shell if needed to update PATH.
opencode --version
opencode service set disabled true
opencode service stop
```

The binary is normally installed at `~/.opencode/bin/opencode`. No Node.js,
pnpm, or executable override is needed when this is the `opencode` in your
`PATH`. If another installation takes precedence, select the curl-installed
binary explicitly:

```bash
export SAFE_OPENCODE_BIN="$HOME/.opencode/bin/opencode"
```

#### Option 2: pnpm

Install a supported [Node.js LTS release](https://nodejs.org/en/download)
and [pnpm](https://pnpm.io/installation), then install OpenCode V2 on the host:

```bash
node --version
pnpm --version
pnpm add -g --allow-build=@opencode/cli @opencode/cli
opencode --version
opencode service set disabled true
opencode service stop
```

If pnpm reports that its global bin directory is missing, run `pnpm setup`
and reopen your shell before installing. The installation's build step installs
and checks the binary; it does not install a running background service.

Point the launcher at the native binary installed by pnpm. Its usual `opencode`
command is a shell shim referring to host paths that are not mounted inside the
sandbox:

```bash
export SAFE_OPENCODE_BIN="$(pnpm root -g)/@opencode/cli/bin/opencode.exe"
```

Despite its `.exe` suffix, this is the native Linux binary on Linux. Set this
variable in each shell where you use the launcher, or add the export to your
shell configuration.

#### Start the server only inside the sandbox

For either installation method, `opencode service set disabled true` disables
automatic background-service startup, and `opencode service stop` stops an
existing service. These setup commands run on the host; launch actual sessions
only through `safe-opencode` for isolation.

Configure your provider and start a session from this repository:

```bash
./safe-opencode /path/to/project -- --standalone
```

Use `/connect` in the sandboxed terminal interface to configure your model
provider. `--standalone` explicitly starts a private server inside the same
Bubblewrap environment instead of connecting to or starting the host background
service. OpenCode V2 binds this server to `127.0.0.1` on an automatically assigned
port, generates an authentication password, and stops it when the client exits.
No separate `opencode serve`, port configuration, or password setup is needed.

**Outbound network access stays enabled**, including access to remote model
providers; a local model is not required. Binding the server to localhost limits
inbound exposure, not outbound requests. The launcher shares the host network
namespace, so localhost is still reachable by other local users and processes.
Do not use `--server` to connect to a host service or start
`serve --service` inside the sandbox for this workflow.

Provider credentials are stored in OpenCode's data directory, which the launcher
exposes read/write to OpenCode. If you override `--config-dir` or
`SAFE_OPENCODE_CONFIG`, the default host service settings are not mounted;
keep using `--standalone` explicitly.

### Usage

By default, the project is mounted read-only at `/workspace`. OpenCode
configuration is read-only, while OpenCode data is writable and persists on the
host. The host home directory is not mounted.

```bash
./safe-opencode /path/to/project -- --standalone
./safe-opencode --rw /path/to/project -- --standalone
```

Additional host directories can be exposed explicitly. Each path must already
exist and be a directory; the script prints its sandbox path before starting
OpenCode.

```bash
./safe-opencode \
  --mount-ro /data/reference \
  --mount-rw /data/results \
  --python-env "$HOME/.venvs/project" \
  /path/to/project -- --standalone
```

Mappings are numbered in the order supplied:

- `--mount-ro PATH` → `/mnt/host-ro/N` (read-only)
- `--mount-rw PATH` → `/mnt/host-rw/N` (read/write)
- `--python-env PATH` → the same absolute path inside the sandbox (read-only,
  with its `bin` directory prepended to `PATH`)

The optional Python environment must contain an executable `bin/python`.
Package installation into that environment will not work because it is mounted
read-only. A virtual environment that depends on other host paths not mounted
into the sandbox may not work.

### Options

- `--rw`: make the project mount writable.
- `--mount-ro PATH`: expose an additional read-only directory.
- `--mount-rw PATH`: expose an additional read/write directory.
- `--python-env PATH`: expose a Python environment read-only.
- `--config-dir PATH` / `--data-dir PATH`: override host OpenCode config/data
  directories.
- `--bwrap PATH`: select a Bubblewrap executable.
- `--no-service-check`: skip the OpenCode service status check.
- `--resume` / `--resume ID`: continue the latest or a selected session.

Config, data, and Bubblewrap paths can also be set with `SAFE_OPENCODE_CONFIG`,
`SAFE_OPENCODE_DATA`, and `SAFE_OPENCODE_BWRAP`.
Set `SAFE_OPENCODE_BIN` to select a native OpenCode executable instead of the
`opencode` command in `PATH`.

The launcher prints a short environment summary that can be pasted into the
opening prompt. For example:

> You are working in a Linux sandbox. The project is at `/workspace`.
> Read reference files from `/mnt/host-ro/1` and save generated results under
> `/mnt/host-rw/1`. Use the configured Python environment's `bin/python`; it is
> read-only. Do not assume other host files or paths are available.

This context is informational only; it is not automatically sent to the model.

## Limitations

- Bubblewrap is Linux-only; this script does not support macOS or Windows.
- Docker is not required, but Bubblewrap and compatible kernel/system
  configuration are.
- Network access is not isolated by the script.
- This is a lightweight, custom hardening tool, not a complete sandbox,
  container replacement, or independently audited security product.
- Some clusters restrict Bubblewrap or user namespaces. Do not try to bypass
  those restrictions; follow local policy.
