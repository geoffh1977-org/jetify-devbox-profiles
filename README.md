# Jetify Devbox Profiles

A small collection of reusable project templates for development environments based on [`geoffh1977/jetify-devbox`](https://hub.docker.com/r/geoffh1977/jetify-devbox). The templates are designed for opening a repository in a Visual Studio Code Dev Container while keeping the project source on the host and the Devbox/Nix caches in named Docker volumes.

The primary audience is developers who want a repeatable container-based Devbox setup without placing language tooling, Nix, or Devbox directly on every host.

## What is included

| Path | Purpose |
| --- | --- |
| `vscode-templates/minimal/` | General-purpose Devbox Dev Container with an empty package list. Use it as the starting point for most projects. |
| `vscode-templates/minimal-with-git/` | General-purpose Devbox Dev Container with an empty package list. Mounts GPG and SSH. Use it as the starting point for most projects. |
| `vscode-templates/docker-image/` | Devbox Dev Container for projects that build Docker images. Includes the Docker CLI, Hadolint, ShellCheck, and pre-commit configuration. It mounts the host Docker socket. |
| `other-templates/console-environment/` | A standalone Compose-based personal/client console environment with isolated home configuration and mounted project and temporary-storage paths. |

Each profile directory has its own README with its capabilities, setup steps, required host tools, optional integrations, and security implications. Start with the profile README when adopting a specific template.

The VS Code templates provide:

- a VS Code Dev Container that uses `geoffh1977/jetify-devbox:latest`;
- the project mounted at `/Project` inside the container;
- a `devbox` user whose UID/GID is configurable to match the host developer;
- persistent named volumes for `/nix`, `/home/devbox/.devbox`, and Bash configuration;
- optional mounts for the host Git configuration, SSH directory, and GPG agent socket; and
- VS Code's integrated **Devbox Shell**, which starts `devbox shell` automatically.

## Host requirements

### Required for the container-based workflow

Install the following on the development host:

1. **Docker Engine** with the Docker Compose v2 plugin, or **Docker Desktop** with Compose support. Docker must be running, and the developer must be permitted to use it.
2. **Visual Studio Code**.
3. The VS Code **Dev Containers** extension (`ms-vscode-remote.remote-containers`).
4. **Git** (strongly recommended) to create/clone repositories and to use the host Git configuration mounted by the templates.

Verify the basic host tooling before starting:

```bash
docker version
docker compose version
code --version
git --version
```

> **No host Devbox installation is required** when using these templates through VS Code Dev Containers: Devbox is supplied by the container image. Docker is the host-side runtime that must be installed.

### Optional host tooling

- **GnuPG / gpg-agent**: required only if Git commits are GPG-signed. The templates expect a Linux-style agent socket under `/run/user/<uid>/gnupg`.
- **Devbox** and **direnv**: useful only when you deliberately want to use a project outside the container. Each template includes `.envrc.example`, which can be copied to `.envrc` to load Devbox through direnv.
- **Docker registry authentication**: required only when the `docker-image` template needs to pull from or push to private registries. The template can mount the host `.docker` directory for this purpose.

### Host identity values

The Dev Container runs as `devbox`, but its UID and GID are configurable. Set them to the host developer's numeric UID and primary GID to avoid root-owned files in the project checkout.

On Linux:

```bash
id -u
id -g
# Required by the docker-image template:
stat -c '%g' /var/run/docker.sock
```

On macOS, use `id -u`, `id -g`, and `stat -f '%g' /var/run/docker.sock`. Docker Desktop environments may have different socket behavior; validate access with `docker ps` before opening the project in the container.

## Create a project from a template

The templates are copied **into the top level of the new project**, including dotfiles. Do not copy just the visible files: `.devcontainer/`, `.vscode/`, and the ignore/configuration files are part of the template.

### 1. Choose a template

- Choose **`minimal`** for a normal Devbox-based project.
- Choose **`docker-image`** when the project builds/lints Dockerfiles or needs to run the host Docker daemon from the development container.

### 2. Copy it to the new project root

With this repository already cloned locally:

```bash
TEMPLATES=/path/to/jetify-devbox-profiles
PROJECT=/path/to/my-project

mkdir -p "$PROJECT"
cp -a "$TEMPLATES/vscode-templates/minimal/." "$PROJECT/"
cd "$PROJECT"
git init
```

For the Docker image template, replace `minimal` with `docker-image`:

```bash
cp -a "$TEMPLATES/vscode-templates/docker-image/." "$PROJECT/"
```

The trailing `/.` is intentional: it copies hidden files and directories into the project root.

### 3. Create local Dev Container settings

The template keeps local host-specific values out of Git. Create the ignored environment file from its example:

```bash
cp .devcontainer/.env.example .devcontainer/.env
```

Edit `.devcontainer/.env` and set:

```dotenv
DEVBOX_UID=1000
DEVBOX_GID=1000
ENVIRONMENT_PATH=/absolute/path/to/your/home-or-development-root
```

`ENVIRONMENT_PATH` must be an **absolute path** to a directory that contains the configuration items you want mounted into the container:

```text
<ENVIRONMENT_PATH>/.config/git/config
<ENVIRONMENT_PATH>/.ssh/
```

The `docker-image` template additionally mounts:

```text
<ENVIRONMENT_PATH>/.docker/
```

For `docker-image`, also set the numeric group ID that owns the host Docker socket:

```dotenv
DOCKER_GID=131
```

Use the `stat` command shown in [Host identity values](#host-identity-values) instead of assuming `131` is correct. The example values are placeholders; do not commit `.devcontainer/.env`.

If you do not use GPG signing, the GPG socket mount is usually harmless on Linux only when the expected path exists. If the path is absent or unsuitable on your platform, adjust or remove the GPG socket environment variables and volume mount in `.devcontainer/devbox.compose.yaml` for that project.

### 4. Open the project in the container

1. Open the project root in VS Code:

   ```bash
   code "$PROJECT"
   ```

2. Run **Dev Containers: Reopen in Container** from the Command Palette.
3. On first start, Docker pulls `geoffh1977/jetify-devbox:latest`, creates the named cache volumes, and starts the `dev` service.
4. Open a new integrated terminal. The provided **Devbox Shell** profile runs `devbox shell` automatically.

You can also test the Compose setup from the project root before using VS Code:

```bash
docker compose --env-file .devcontainer/.env \
  -f .devcontainer/devbox.compose.yaml config
```

This validates the rendered Compose configuration without starting the development service.

## Configure project dependencies

`devbox.json` is the project manifest. Add only the runtimes, tools, and CLIs required by the project. From a terminal inside the Dev Container:

```bash
cd /Project
devbox add nodejs@22
# or another package appropriate to the project
devbox shell
```

Devbox updates `devbox.json` and generates `devbox.lock`. The supplied `.gitignore` intentionally ignores `devbox.lock`, `devbox.d/`, and `.devbox/`; follow that convention unless the project has a specific reason to commit a lock file.

Useful Devbox commands inside the container:

```bash
cd /Project
devbox shell
devbox run -- <command>
devbox update
devbox rm <package>
```

The supplied manifests do **not** define named `shell.scripts`, so use explicit commands such as `devbox run -- <command>` rather than expecting a project-specific shortcut to exist. Add named scripts to `devbox.json` only when the project needs a stable, documented workflow.

### Optional direnv integration

For developers using the project directly on a host with Devbox and direnv installed:

```bash
cp .envrc.example .envrc
direnv allow
```

The generated `.envrc` evaluates `devbox generate direnv --print-envrc`. This is optional for the Dev Container workflow because the VS Code terminal profile already enters `devbox shell`.

## Template-specific notes

### `minimal`

`minimal` starts with no Devbox packages. It is the right base for language-specific projects, infrastructure repositories, scripts, or any project that needs to define its own tooling. Add dependencies to `devbox.json` after copying the template.

Its VS Code configuration installs the `mkhl.direnv` extension and trusts Jetify's Devbox JSON schema URL.

### `docker-image`

Use this template for Docker image projects. Its Devbox manifest currently provides:

- `docker-client`
- `hadolint`
- `shellcheck`
- `pre-commit`

It also configures VS Code extensions for EditorConfig, Hadolint, Dev Containers, direnv, and ShellCheck.

The included pre-commit configuration validates YAML and JSON, fixes trailing whitespace/end-of-file issues, checks EditorConfig rules, runs ShellCheck on shell files, and runs Hadolint for `docker/Containerfile`.

Install the hooks after copying the template:

```bash
cd /Project
pre-commit install
pre-commit run --all-files
```

### Docker socket security

The `docker-image` template bind-mounts `/var/run/docker.sock`. Anyone with shell access inside that Dev Container can control the host Docker daemon; on typical Linux Docker hosts, that is effectively privileged host access. Use this template only for trusted projects and trusted developers. Prefer `minimal` when Docker daemon access is unnecessary.

The Compose configuration also grants `SYS_ADMIN` and uses `seccomp:unconfined`. Treat the container as a trusted local development environment, not as a hardened sandbox.

## Git, SSH, GPG, and registry credentials

The Dev Container mounts selected host configuration so normal developer workflows can work without copying private keys or configuration into the image:

- Git config: mounted read-only from `${ENVIRONMENT_PATH}/.config/git/config`
- SSH directory: mounted read-only from `${ENVIRONMENT_PATH}/.ssh`
- GPG agent directory: mounted read-only from `/run/user/${DEVBOX_UID}/gnupg`
- Docker credentials/configuration (`docker-image` only): mounted from `${ENVIRONMENT_PATH}/.docker`

These mounts are convenient but sensitive. Keep `ENVIRONMENT_PATH` local, ensure it points only at the intended developer configuration, and never add private credentials, local `.env` files, or host-specific paths to source control.

If the project does not need one of these integrations, remove its mount from `.devcontainer/devbox.compose.yaml` rather than mounting more host data than necessary.

## Troubleshooting

### Docker permission denied

Confirm Docker works for the host user first:

```bash
docker ps
```

On Linux, add the user to the appropriate Docker group according to your distribution's Docker installation guidance, then start a new login session. For the `docker-image` template, also confirm `DOCKER_GID` matches the group ID of `/var/run/docker.sock`.

### Files are owned by the wrong user

Check the host values and make them match the developer account:

```bash
id -u
id -g
```

Update `DEVBOX_UID` and `DEVBOX_GID` in `.devcontainer/.env`, then use **Dev Containers: Rebuild Container**. The template's `init-devbox-volumes` service repairs ownership of the persistent Devbox and Bash cache volumes before the development service starts.

### A mounted Git, SSH, Docker, or GPG path does not exist

Docker Compose cannot mount a missing host path reliably. Either create/configure the expected host component or remove/adjust its corresponding mount in `.devcontainer/devbox.compose.yaml`. This is particularly relevant for GPG sockets and Docker Desktop hosts.

### Devbox packages are not available

Open a new integrated terminal, which should use **Devbox Shell**, or run:

```bash
cd /Project
devbox shell
```

After changing the manifest, run `devbox update` or rebuild the container if you need a clean environment refresh.

## Maintaining this repository

The repository root itself uses Devbox to provide `pre-commit`. From the repository root:

```bash
devbox run -- pre-commit run --all-files
```

When adding a template, keep it self-contained under `vscode-templates/<template-name>/`, include required dotfiles, avoid committing local `.env` files or secrets, and update this README with its intended use, dependencies, host mounts, and security considerations.
