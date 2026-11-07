# safe-code

`safe-code` is a custom implementation of filesystem sandboxing for AI coding
tools, using [Bubblewrap](https://github.com/containers/bubblewrap) on Linux.
It is intended to make it easier to run tools such as OpenCode with less access
to the host filesystem than they would normally have.

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

## Requirements

- Linux.
- Bubblewrap (`bwrap`), installed and usable by the current account. The host
  must allow user namespaces or provide an appropriately configured Bubblewrap
  installation.
- OpenCode available as `opencode` in the host `PATH`.
- `bash`, `realpath`, and `readlink`.

The script creates host OpenCode config and data directories if they do not
exist. Defaults are `${XDG_CONFIG_HOME:-$HOME/.config}/opencode` and
`${XDG_DATA_HOME:-$HOME/.local/share}/opencode`.

## `safe-opencode`

By default, the project is mounted read-only at `/workspace`. OpenCode
configuration is read-only, while OpenCode data is writable and persists on the
host. The host home directory is not mounted.

```bash
./safe-opencode /path/to/project
./safe-opencode --rw /path/to/project
```

Additional host directories can be exposed explicitly. Each path must already
exist and be a directory; the script prints its sandbox path before starting
OpenCode.

```bash
./safe-opencode \
  --mount-ro /data/reference \
  --mount-rw /data/results \
  --python-env "$HOME/.venvs/project" \
  /path/to/project
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
