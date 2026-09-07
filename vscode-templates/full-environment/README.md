# Full Environment VS Code Profile

A VS Code Dev Container template for projects that need the same full Zsh, Starship, Git, SSH, Docker, and Devbox experience as the [`console-environment`](../../other-templates/console-environment/) profile.

It can run on its own, or alongside a console environment. When both profiles use the same absolute `ENVIRONMENT_PATH`, they deliberately use the same host-backed developer configuration and credentials without copying them into either image.

## What this profile provides

- A `geoffh1977/jetify-devbox:latest` Dev Container with the project mounted at `/Project`.
- A project-level `devbox.json` that loads the `zsh-environment` plugin and provides `docker-client` and `go-task`.
- A VS Code integrated terminal that starts `devbox shell` through the project Zsh configuration.
- Persistent named volumes for the Nix store, the project's Devbox state, Devbox configuration, the VS Code Server, and user-share data.
- Host-mounted Git, SSH, Docker, Zsh, Starship, temporary-storage, Docker-socket, and optional GPG-agent integrations.
- VS Code extensions for direnv, OpenTofu, and GitHub Actions.

This is a **project Dev Container template**. Copy it into a project root, including its dot-directories; it is not intended to be opened directly from this template repository.

## Relationship to `console-environment`

The two profiles are complementary:

| Concern | Full environment VS Code profile | Console environment |
| --- | --- | --- |
| Primary use | Open one project in VS Code Dev Containers | Reusable terminal-first client console |
| Source location | Current project directory → `/Project` | `Projects/` directory → `/home/devbox/Projects` |
| Active Devbox manifest | Project `devbox.json` | `${ENVIRONMENT_PATH}/devbox.json` |
| Zsh startup file | `${ENVIRONMENT_PATH}/.config/zsh/zshrc-project` | `${ENVIRONMENT_PATH}/.config/zsh/zshrc-client` |
| Shared configuration | `${ENVIRONMENT_PATH}/.ssh` and selected `.config` paths | The same paths when configured with the same `ENVIRONMENT_PATH` |
| Persistent caches | Its own Compose named volumes | Its own Compose named volumes |

Using the same `ENVIRONMENT_PATH` therefore shares **configuration state**—such as Git identity, SSH keys, Docker registry login, Zsh configuration, and Starship prompt settings—not the containers' Nix stores, Devbox state, VS Code Server caches, or project manifests.

A typical paired setup is:

```text
$HOME/Development/Personal/Home     # ENVIRONMENT_PATH for both profiles
$HOME/Development/Personal/Temp     # TEMP_PATH for both profiles
$HOME/Development/Projects/example  # project created from this template
```

The console's `Home/devbox.json` remains its home-level console manifest. This template's `devbox.json` remains the project-level manifest and may contain project-specific tools such as `go-task`.

## Host prerequisites

Install and verify:

1. Docker Engine with Docker Compose v2, or Docker Desktop with Compose support.
2. Visual Studio Code and the **Dev Containers** extension (`ms-vscode-remote.remote-containers`).
3. Git, recommended for source control and the mounted Git configuration.
4. A Docker socket at `/var/run/docker.sock` if Docker commands will be used in the container.
5. The host paths mounted by `.devcontainer/devbox.compose.yaml`.

```bash
docker version
docker compose version
code --version
git --version
id -u
id -g
stat -c '%g' /var/run/docker.sock
```

On macOS, use `stat -f '%g' /var/run/docker.sock` for the Docker socket group. Docker Desktop can present socket access differently, so confirm `docker ps` works for the host user before opening the Dev Container.

### Required environment paths

`ENVIRONMENT_PATH` is an absolute, local path. The current Compose configuration mounts:

```text
<ENVIRONMENT_PATH>/.ssh/
<ENVIRONMENT_PATH>/.config/docker/
<ENVIRONMENT_PATH>/.config/git/
<ENVIRONMENT_PATH>/.config/zsh/
<ENVIRONMENT_PATH>/.config/zsh/zshrc-project
<ENVIRONMENT_PATH>/.config/starship.toml
```

It also mounts the host GPG-agent directory from `/run/user/<DEVBOX_UID>/gnupg` and `TEMP_PATH` at `/home/devbox/Temp`. If an integration is not used or the required source path does not exist, remove or adapt its environment variables and mount in `.devcontainer/devbox.compose.yaml` before opening the container. Docker Compose cannot reliably bind-mount an absent source path.

## Create a project from this template

### 1. Copy every template file into the project root

Copy hidden files and directories as well as ordinary files:

```bash
TEMPLATES=/path/to/jetify-devbox-profiles
PROJECT=/path/to/my-project

mkdir -p "$PROJECT"
cp -a "$TEMPLATES/vscode-templates/full-environment/." "$PROJECT/"
cd "$PROJECT"
git init
```

The trailing `/.` is intentional. It includes `.devcontainer/`, `.vscode/`, `.envrc.example`, and `.gitignore`.

### 2. Create local Dev Container settings

Create the ignored local environment file:

```bash
cp .devcontainer/.env.example .devcontainer/.env
```

Set values for the actual host developer and paths:

```dotenv
CLIENT_NAME=my-project
DEVBOX_UID=1000
DEVBOX_GID=1000
DOCKER_GID=131
ENVIRONMENT_PATH=/absolute/path/to/environment-home
TEMP_PATH=/absolute/path/to/temporary-storage
```

Use `id -u`, `id -g`, and the `stat` command above rather than assuming the example numeric values are correct. Do not commit `.devcontainer/.env`.

To pair this template with a console environment, point `ENVIRONMENT_PATH` at the console profile's `Home/` directory and, if desired, point `TEMP_PATH` at its `Temp/` directory. For example:

```dotenv
ENVIRONMENT_PATH=/path/to/console-environment/Home
TEMP_PATH=/path/to/console-environment/Temp
```

### 3. Validate and open the container

From the project root, validate the rendered configuration before starting the Dev Container:

```bash
docker compose --env-file .devcontainer/.env \
  -f .devcontainer/devbox.compose.yaml config
```

Then open the project in VS Code and run **Dev Containers: Reopen in Container**. The configuration starts the `init-volumes` service first to align managed-volume ownership with `DEVBOX_UID` and `DEVBOX_GID`, then starts the `dev` service as the `devbox` user.

Open a new integrated terminal after startup. The mounted `zshrc-project` starts `devbox shell` when VS Code opens a terminal outside an existing Devbox shell. The first shell initializes the project Devbox environment and makes the configured Zsh binary available at `/Project/.devbox/nix/profile/default/bin/zsh`.

## Project dependencies

The supplied `devbox.json` contains the shared Zsh plugin plus `docker-client` and `go-task`. Add only project-specific runtimes and tools from a terminal in the container:

```bash
cd /Project
devbox add nodejs@22
devbox shell
```

Useful commands:

```bash
cd /Project
devbox shell
devbox run -- <command>
devbox update
devbox rm <package>
```

`devbox.lock`, `devbox.d/`, and `.devbox/` are ignored by the supplied `.gitignore`. The optional `.envrc.example` supports host-side Devbox and direnv usage, but is not required for the VS Code Dev Container workflow.

## Security boundaries

This is a trusted local development environment, **not** a hardened sandbox.

- The Dev Container mounts the host Docker socket; access to it is commonly equivalent to privileged control of the host Docker daemon.
- It adds `SYS_ADMIN` and uses `seccomp:unconfined` for Docker compatibility.
- SSH, Git, Docker, Zsh, and Starship configuration are mounted from `ENVIRONMENT_PATH`. The Compose file does not mark those mounts read-only, so code running in the container can alter them.
- The GPG-agent directory is mounted read-only, but using the agent can still authorize signatures according to the host agent's policy.

Use this template only with trusted projects and trusted images. Keep one environment directory per trust boundary or client context, minimize mounts to only what the project needs, remove unused integrations, and never commit private keys, registry credentials, GPG material, or local `.env` files.

## Troubleshooting

### Zsh is unavailable in the integrated terminal

The `SHELL` value points to the Zsh binary created by the project Devbox profile. On the first start, open an integrated terminal and allow `devbox shell` to initialize the profile. If needed, run:

```bash
cd /Project
devbox shell
```

If the profile was changed or initialisation did not complete, run **Dev Containers: Rebuild Container** and open a new terminal.

### Docker permission denied

First verify host access:

```bash
docker ps
```

On Linux, ensure the host user belongs to the group that owns the Docker socket, start a new login session, and set `DOCKER_GID` in `.devcontainer/.env` to the socket's actual group ID.

### A configuration or credential path cannot be mounted

Create the required non-secret configuration path locally, or remove/adapt the corresponding mount in `.devcontainer/devbox.compose.yaml`. Do not work around a missing path by committing a credential or host-specific configuration file into the project.
