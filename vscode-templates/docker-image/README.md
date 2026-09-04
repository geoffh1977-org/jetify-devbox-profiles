# Docker Image VS Code Devbox Profile

A Visual Studio Code Dev Container profile for repositories that build, lint, and publish container images. It runs Devbox inside `geoffh1977/jetify-devbox:latest`, keeps the repository on the host at `/Project`, and deliberately connects the container to the host Docker daemon.

## What this profile provides

- Everything in the minimal Devbox Dev Container profile: a VS Code Dev Container, automatic `devbox shell`, configurable UID/GID, and persistent Nix/Devbox caches.
- Devbox packages for `go-task`, `docker-client`, `hadolint`, `shellcheck`, and `pre-commit`.
- VS Code extensions for EditorConfig, Hadolint, Dev Containers, direnv, and ShellCheck.
- A host Docker-socket mount so Docker commands inside the Dev Container use the host daemon.
- A mount for host Docker configuration (`<ENVIRONMENT_PATH>/.docker`) for registry authentication when needed.
- Example `Taskfile` and pre-commit configuration for Docker-image build and quality-check workflows.

Copy `taskfile.yaml.example` to `taskfile.yaml` and replace its `IMAGE_USERNAME` and `IMAGE_NAME` values before using its build or push tasks.

## Required host requirements

Install and verify these on the development host:

1. **Docker Engine with Docker Compose v2**, or **Docker Desktop with Compose support**. Docker must be running and accessible to the developer.
2. **Visual Studio Code** with the **Dev Containers** extension (`ms-vscode-remote.remote-containers`).
3. A Docker socket at `/var/run/docker.sock` that the container can use, and the numeric group ID that owns it.
4. A local `ENVIRONMENT_PATH` containing the required Git, SSH, and Docker configuration paths—or a deliberate Compose-file edit removing unneeded mounts.

```bash
docker version
docker compose version
docker ps
code --version
id -u
id -g
stat -c '%g' /var/run/docker.sock
```

On macOS, obtain the socket group with `stat -f '%g' /var/run/docker.sock`. Docker Desktop socket behaviour varies; confirm access with `docker ps` before opening the Dev Container.

No host installation of Devbox, Nix, Hadolint, ShellCheck, pre-commit, Task, or the Docker CLI is required for the container workflow; those tools are supplied inside the profile.

### Host identity and configuration

Create the ignored local environment file:

```bash
cp .devcontainer/.env.example .devcontainer/.env
```

Set `DEVBOX_UID`, `DEVBOX_GID`, `DOCKER_GID`, and absolute `ENVIRONMENT_PATH` values. The supplied Compose configuration expects:

```text
<ENVIRONMENT_PATH>/.config/git/config
<ENVIRONMENT_PATH>/.ssh/
<ENVIRONMENT_PATH>/.docker/
```

The default GPG socket mount assumes Linux at `/run/user/<uid>/gnupg`. Remove or adapt it if signed commits are not required or the host uses a different layout. Never commit `.devcontainer/.env`.

## Open and use the profile

1. Copy this directory, including dotfiles, into the target project root.
2. Configure `.devcontainer/.env`.
3. Open the project in VS Code and choose **Dev Containers: Reopen in Container**.
4. Open a new integrated terminal; it starts in `devbox shell`.

Validate Compose before starting the container:

```bash
docker compose --env-file .devcontainer/.env \
  -f .devcontainer/devbox.compose.yaml config
```

Install and run the supplied checks from `/Project` after copying the template:

```bash
pre-commit install
pre-commit run --all-files
```

## Optional but recommended host tools

- **Git**: recommended for repository work and the mounted Git configuration.
- **GnuPG and `gpg-agent`**: only for GPG-signed commits.
- **Docker registry login**: required only to pull from or push to private registries; authenticate the host Docker configuration that is mounted at `.docker`.
- **Devbox and direnv**: useful only when working outside the Dev Container. Copy `.envrc.example` to `.envrc` and run `direnv allow` after installing both.

## Security notes

This profile bind-mounts `/var/run/docker.sock`. Any process in the Dev Container can control the host Docker daemon, which is effectively privileged host access on typical Linux systems. It also mounts SSH keys and Docker credentials and runs with `SYS_ADMIN` plus `seccomp:unconfined`. Use it only for trusted local projects and trusted developers. Use the `minimal` profile when a project does not need Docker-daemon access.
