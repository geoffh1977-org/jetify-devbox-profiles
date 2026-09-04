# Console Environment Profile

A standalone Docker Compose profile for running a reusable Devbox console with an isolated, bind-mounted home environment. It is intended for one or more distinct local client contexts—such as separate customer or personal environments—without mixing their SSH keys, Docker configuration, shell setup, Devbox manifest, or project directories.

This is a terminal-focused Compose profile, not a VS Code Dev Container template.

## What this profile provides

- A `geoffh1977/jetify-devbox:latest` console container that opens `devbox shell` as the `devbox` user.
- Separate host-backed directories mounted as:
  - `Home/` → the container's Devbox home configuration and `devbox.json`/`devbox.lock`.
  - `Projects/` → `/home/devbox/Projects`.
  - `Temp/` → `/home/devbox/Temp`.
- Persistent named Docker volumes for the Nix store, Devbox state, VS Code Server cache, user cache, and user-share data.
- A startup initializer that fixes ownership of managed Docker volumes to the configured host UID/GID.
- Docker socket access, allowing Docker commands in the console to control the host Docker daemon.
- A `Taskfile` that generates the local Compose `.env` file and provides lifecycle commands.

The supplied `Home/devbox.json` loads the `zsh-environment` Devbox plugin and includes `docker-client`. Customize it for the intended environment; keep secrets and private credentials out of source control.

## Required host requirements

This profile is designed primarily for a Linux host. Install and verify:

1. **Docker Engine with the Docker Compose v2 plugin**. Docker must be running and accessible to the developer.
2. **Task** (`go-task`) on the host. The `Taskfile` is the supported lifecycle interface and runs before the container starts.
3. Standard Linux utilities used by `task generate-compose-env`: `id`, `getent`, `cut`, `basename`, `tr`, `grep`, `clear`, and `rm`.
4. A Docker socket at `/var/run/docker.sock`; the profile mounts it into the console.
5. The `Home/`, `Projects/`, and `Temp/` directories in this profile, including the paths mounted by `compose.yaml`.

```bash
docker version
docker compose version
docker ps
task --version
id -u
id -g
getent group docker
stat -c '%g' /var/run/docker.sock
```

The Taskfile derives `DOCKER_GID` from the host `docker` group, so that group must exist. If the Docker socket is owned by another group, adjust the generated `.env` value or the Taskfile deliberately rather than assuming they match.

### Required host paths

The current Compose configuration mounts these host paths from `Home/`:

```text
Home/.ssh/
Home/devbox.json
Home/devbox.lock
Home/.config/docker/
Home/.config/zsh/
Home/.config/zsh/zshrc-client
Home/.config/starship.toml
```

It also mounts `/run/user/<DEVBOX_UID>/gnupg` and `/etc/zoneinfo/Australia/Melbourne`. Create or configure the needed paths, or remove/adapt their Compose mounts before startup. The GPG path is needed only for signing; it should not be mounted unnecessarily.

## Start and operate the console

Run commands from this directory. The Taskfile generates `.env` automatically for commands that need it.

```bash
# Show the available lifecycle commands
task

# Start the reusable console in the background
task up

# Open or attach to a Devbox shell
task terminal

# Stop it
task down
```

On first use, initialize the managed volumes explicitly:

```bash
task init-volumes
task up
task terminal
```

Other lifecycle commands are `task restart` and `task nuke`. `task nuke` runs `docker compose down -v` and deletes the generated `.env`; it permanently removes this profile's named volumes. Do not run it unless discarding the cached Nix, Devbox, VS Code Server, and user-cache state is intended.

Validate the rendered Compose configuration without starting a service:

```bash
task generate-compose-env
docker compose config
```

## Optional but recommended host tools

- **Git**: recommended for managing the repositories mounted under `Projects/`.
- **GnuPG with a running `gpg-agent`**: only if signing Git commits. The provided socket mount assumes a Linux user-runtime layout.
- **Docker registry authentication**: only when the console pulls from or pushes to private registries. Store its Docker client configuration under `Home/.config/docker/`.
- **VS Code with Remote/Dev Containers tooling**: optional if you want editor access to the mounted projects; the profile itself does not require VS Code.
- **Devbox and direnv on the host**: optional. The container image already supplies Devbox, so host installations are only useful for intentionally working outside the console container.

## Security notes

The console is a trusted local environment, not a sandbox. It mounts the host Docker socket, SSH directory, optional GPG-agent socket, and Docker configuration; processes in the container can therefore act with the permissions those integrations grant. Keep separate client contexts in separate copies of this profile, minimize the mounted host data, use trusted images and projects only, and never commit the generated `.env` or credential material.
