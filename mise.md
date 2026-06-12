# mise

**What it is:** A single tool that manages **dev tool versions** (Node, Python, Go, …), **environment variables**, and **tasks** per project. Pronounced "meez" (as in *mise en place*); formerly `rtx`.

**Why people use it:** Replaces `nvm` + `pyenv` + `rbenv` + `direnv` + a Makefile with one fast (Rust) tool. `cd` into a project and the right tool versions + env vars activate automatically.

**Typically used for:** Pinning per-project language/tool versions, auto-loading `.env`-style config on directory entry, and as a lightweight task runner.

---

## Mental model

- **`mise.toml`** (or `.mise.toml`) = per-project config: tools, env, tasks. Commit it.
- mise reads it on `cd`, installs/activates the pinned versions, and shims the binaries onto your `PATH`.
- Backed by **asdf-compatible** plugins plus fast core providers for popular languages.

---

## Setup

```bash
curl https://mise.run | sh                 # install
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc   # hook into shell (bash/fish too)
mise doctor                                # diagnose install/activation issues
```

## Most common commands

```bash
mise use node@20            # install + pin node 20 in this project (writes mise.toml)
mise use -g node@lts        # set a global default
mise install                # install everything in mise.toml
mise ls                     # list installed tools + versions
mise ls-remote node         # list installable versions
mise current                # show active versions for this dir
mise exec -- node app.js    # run a command with mise's tools on PATH
mise run <task>             # run a defined task
mise upgrade                # upgrade installed tools
mise self-update            # update mise itself
```

## Simple usage

Pin tools for a project:

```bash
mise use node@20 python@3.12
```

Produces:

```toml
# mise.toml
[tools]
node = "20"
python = "3.12"
```

Now anyone who clones the repo runs `mise install` and gets identical versions.

---

## Advanced

### Environment variables + secrets

```toml
# mise.toml
[env]
NODE_ENV = "development"
DATABASE_URL = "postgres://localhost/app"
_.file = ".env"                 # load vars from a dotenv file
_.path = ["./node_modules/.bin"]  # prepend to PATH
```

Vars load on `cd` and unload on leave — a `direnv` replacement with no extra tool.

### Tasks (a Make replacement)

```toml
[tasks.test]
run = "npm test"

[tasks.dev]
run = "npm run dev"
env = { PORT = "3000" }

[tasks.ci]
depends = ["lint", "test"]      # run lint + test first
run = "npm run build"
```

```bash
mise run dev
mise run ci
mise tasks            # list available tasks
```

File-based tasks also work: drop an executable script in `mise-tasks/deploy` and call `mise run deploy`.

### Pin exact versions for reproducibility

```bash
mise use node@20.11.1     # exact patch, not a floating major
```

`mise.lock` (enable with `mise settings lockfile=true`) locks resolved versions for fully reproducible installs across a team.

> **Grounded:** mise is the actively-maintained successor to `rtx` and is built on the [asdf](https://asdf-vm.com/) plugin ecosystem, so any asdf plugin works. See the [official docs](https://mise.jdx.dev/). It's increasingly used in CI (GitHub Actions has `jdx/mise-action`) as a single step to provision all tool versions a repo needs.

### Use in CI (GitHub Actions)

```yaml
- uses: jdx/mise-action@v2      # installs everything in mise.toml
- run: mise run ci
```

---

## Practical recipes

**Install every tool a freshly-cloned repo needs:**
```bash
mise install
```

**Switch a project to a new Node major and update the config in one step:**
```bash
mise use node@22
```

**Run a command with project tools without activating the shell hook:**
```bash
mise exec node@20 -- node --version
```

**See exactly which binary/version will run:**
```bash
mise which node && mise current
```

**Reload env/tools after editing mise.toml (without leaving the dir):**
```bash
mise trust && mise install
```

**Trust a new project's config (mise blocks untrusted configs by default):**
```bash
mise trust
```

**Prune tool versions no longer referenced by any config (reclaim disk):**
```bash
mise prune
```

**Global default toolset for new/unconfigured dirs:**
```bash
mise use -g node@lts python@3.12 go@latest
```

**List outdated tools and upgrade:**
```bash
mise outdated && mise upgrade
```

---

## Tips

- Commit `mise.toml`; it's the per-project source of truth.
- Prefer exact versions (or a lockfile) for reproducibility; floating majors drift.
- mise must be **activated in your shell** (`mise activate`) for auto-switching; in scripts/CI use `mise exec` or `mise-action`.
- New configs are untrusted until `mise trust` — a guard against running arbitrary env from cloned repos.
