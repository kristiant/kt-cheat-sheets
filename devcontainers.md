# Dev Containers

**What it is:** A spec (`.devcontainer/devcontainer.json`) that defines a **full, containerized development environment** — the image, tools, extensions, and settings your project needs to be worked on.

**Why people use it:** Onboarding becomes "open the repo, reopen in container" — everyone gets the identical toolchain, no "install these 12 things first" README. The container *is* the dev environment.

**Typically used for:** Reproducible team dev environments, VS Code / GitHub Codespaces, isolating project toolchains from the host, and CI that mirrors local dev exactly.

---

## Mental model

- **`devcontainer.json`** = the recipe. Points at an image (or Dockerfile / Compose), adds **features**, editor settings, and lifecycle hooks.
- **Features** = reusable, composable installers (Node, Docker, AWS CLI…) you bolt on without writing Dockerfile lines.
- Runs anywhere that speaks the spec: VS Code, Codespaces, the `devcontainer` CLI, JetBrains.

---

## Simple usage

Minimal `.devcontainer/devcontainer.json`:

```jsonc
{
  "name": "My App",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:20",
  "forwardPorts": [3000],
  "postCreateCommand": "npm install"
}
```

In VS Code: **Cmd+Shift+P → "Dev Containers: Reopen in Container"**. Done — you're now editing inside the container.

## Most common fields

```jsonc
{
  "name": "My App",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "features": { ... },             // bolt-on tools (see below)
  "forwardPorts": [3000, 5432],    // expose container ports to host
  "postCreateCommand": "npm ci",   // run once after container is built
  "postStartCommand": "npm run dev",
  "remoteUser": "node",            // non-root user inside the container
  "customizations": {
    "vscode": {
      "extensions": ["dbaeumer.vscode-eslint", "esbenp.prettier-vscode"],
      "settings": { "editor.formatOnSave": true }
    }
  }
}
```

---

## Advanced

### Features (compose tools without Dockerfile edits)

```jsonc
{
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/node:1": { "version": "20" },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/aws-cli:1": {}
  }
}
```

> **Grounded:** The Dev Container spec is an [open standard](https://containers.dev) (Microsoft, now community-governed) used by **GitHub Codespaces** — push a `devcontainer.json` and Codespaces builds the exact environment in the cloud. The [feature registry](https://containers.dev/features) is the canonical catalogue of bolt-on installers.

### Custom Dockerfile

```jsonc
{
  "build": {
    "dockerfile": "Dockerfile",
    "args": { "VARIANT": "20" }
  }
}
```

### Multi-service via Docker Compose

When your app needs a DB/cache alongside it, point the dev container at a Compose service:

```jsonc
{
  "dockerComposeFile": "../compose.yaml",
  "service": "app",
  "workspaceFolder": "/workspace",
  "shutdownAction": "stopCompose"
}
```

Your editor attaches to the `app` service; `db`, `redis`, etc. come up with it.

### Lifecycle hooks (run order)

```jsonc
{
  "initializeCommand": "echo runs on HOST before build",
  "onCreateCommand": "runs once, during image creation (cacheable)",
  "postCreateCommand": "npm ci",        // once, after create — deps go here
  "postStartCommand": "npm run dev",    // every time the container starts
  "postAttachCommand": "echo editor attached"
}
```

### Mounts & secrets

```jsonc
{
  "mounts": [
    "source=${localEnv:HOME}/.aws,target=/home/node/.aws,type=bind,readonly"
  ],
  "remoteEnv": { "NODE_ENV": "development" }
}
```

---

## Practical recipes

**Build & run a dev container from the CLI (no editor needed):**
```bash
npm install -g @devcontainers/cli
devcontainer up --workspace-folder .
```

**Run a command inside the dev container from the CLI (great for CI):**
```bash
devcontainer exec --workspace-folder . npm test
```

**Rebuild after changing devcontainer.json (VS Code):**
```
Cmd+Shift+P → "Dev Containers: Rebuild Container"
```

**Rebuild without cache (when a feature/Dockerfile change isn't picked up):**
```
Cmd+Shift+P → "Dev Containers: Rebuild Without Cache"
```

**Open the current folder in a container in one command:**
```
Cmd+Shift+P → "Dev Containers: Reopen in Container"
```

**Spin one up in the cloud (GitHub Codespaces):**
```bash
gh codespace create --repo owner/repo
gh codespace ssh
```

**Quick base images to start from (Microsoft registry):**
```
mcr.microsoft.com/devcontainers/base:ubuntu          # generic
mcr.microsoft.com/devcontainers/typescript-node:20   # TS/Node
mcr.microsoft.com/devcontainers/python:3.12          # Python
mcr.microsoft.com/devcontainers/go:1.22              # Go
```

---

## Tips

- Put dependency installs in `postCreateCommand` (runs once), not `postStartCommand` (runs every start).
- Pin image tags and feature versions for reproducibility.
- Use `remoteUser` (non-root) — matches prod and avoids root-owned files leaking onto the host via mounts.
- The `devcontainer` CLI lets you reuse the exact same environment in CI as developers use locally.
- Bind-mount host credentials (`~/.aws`, `~/.ssh`) read-only rather than copying secrets into the image.
