# Minimal VS Code Devbox Profile

A deliberately small Visual Studio Code Dev Container starting point for a project that wants Devbox without an opinionated language stack. It uses `geoffh1977/jetify-devbox:latest`; Devbox and Nix run **inside** the container, while the repository remains on the host at `/Project`.

## What this profile provides

- A VS Code Dev Container whose default terminal starts `devbox shell`.
- An empty `devbox.json`, ready for the project to define exactly the packages it needs.
- Persistent Docker volumes for the Nix store and the Devbox user's cache.
- Configurable `DEVBOX_UID` and `DEVBOX_GID` so files created in the bind-mounted checkout match the host developer.
- Read-only mounts for host Git configuration and SSH keys, plus an optional Linux GPG-agent socket mount for signed commits.
- VS Code configuration for the `mkhl.direnv` extension and Devbox JSON schema support.

Choose this profile for ordinary application, scripting, infrastructure, or language-specific repositories. Add the required project packages after copying it:

```bash
cd /Project
devbox add <package>
devbox shell
```

## Required host requirements

Install and verify these on the development host before opening the project:

1. **Docker Engine with Docker Compose v2**, or **Docker Desktop with Compose support**. The Docker daemon must be running and usable by the developer.
2. **Visual Studio Code** with the **Dev Containers** extension (`ms-vscode-remote.remote-containers`).
3. A local directory containing the Git and SSH paths referenced by `.devcontainer/.env`, or an intentional edit to remove those mounts from `.devcontainer/devbox.compose.yaml`.

```bash
docker version
docker compose version
code --version
docker ps
```

No host Devbox, Nix, language runtime, or compiler installation is required for the Dev Container workflow.

### Host identity and configuration

Copy the local environment example, then set values that belong to the host developer:

```bash
cp .devcontainer/.env.example .devcontainer/.env
id -u
id -g
```

Set `DEVBOX_UID`, `DEVBOX_GID`, and an absolute `ENVIRONMENT_PATH` in `.devcontainer/.env`. The current Compose file expects these paths to exist:

```text
<ENVIRONMENT_PATH>/.config/git/config
<ENVIRONMENT_PATH>/.ssh/
```

If a path is not wanted or does not exist, remove or adjust its corresponding volume mount before reopening the container. Do not commit `.devcontainer/.env`.

## Open the profile

1. Copy this directory, including hidden files, to the root of a new or existing project.
2. Create and configure `.devcontainer/.env` as shown above.
3. Open the project root in VS Code and run **Dev Containers: Reopen in Container**.
4. Open a new integrated terminal; the **Devbox Shell** terminal profile enters `devbox shell` automatically.

To validate the Compose definition before starting it, run this from the project root:

```bash
docker compose --env-file .devcontainer/.env \
  -f .devcontainer/devbox.compose.yaml config
```

## Optional but recommended host tools

- **Git**: recommended for cloning projects and using the mounted Git configuration.
- **GnuPG with a running `gpg-agent`**: only for GPG-signed Git commits. The supplied mount assumes Linux at `/run/user/<uid>/gnupg`; adjust or remove it on macOS, Windows, or another unsupported layout.
- **Devbox and direnv**: only for deliberately using the repository outside the container. Copy `.envrc.example` to `.envrc` and run `direnv allow` after installing both tools.

## Security notes

The profile exposes the selected host Git configuration and SSH directory to the container. Keep `ENVIRONMENT_PATH` narrow, use only trusted repositories, and remove mounts the project does not need. The Dev Container also uses `SYS_ADMIN` and `seccomp:unconfined` for compatibility, so it is a trusted development environment—not a hardened isolation boundary.
