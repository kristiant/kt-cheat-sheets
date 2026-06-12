# GitHub Actions

**What it is:** GitHub's built-in CI/CD. Workflows defined in YAML (`.github/workflows/`) run on events (push, PR, schedule) on GitHub-hosted or self-hosted runners.

**Why people use it:** CI/CD that lives in the repo, no external service, a huge marketplace of reusable **actions**, and tight integration with PRs, secrets, and environments.

**Typically used for:** Running tests/lint on PRs, building and publishing artifacts, deploying, releasing packages, and scheduled jobs.

---

## Anatomy

```yaml
# .github/workflows/ci.yml
name: CI
on:                          # triggers
  push:
    branches: [main]
  pull_request:

jobs:
  test:                      # a job runs on its own runner
    runs-on: ubuntu-latest
    steps:                   # steps run sequentially in the same runner
      - uses: actions/checkout@v4          # a reusable action
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci                        # a shell command
      - run: npm test
```

- **Workflow** → **jobs** (parallel by default) → **steps** (sequential).
- `uses:` runs a published action; `run:` runs a shell command.

---

## Common triggers

```yaml
on:
  push:
    branches: [main]
    tags: ["v*"]
    paths: ["src/**"]              # only when these files change
  pull_request:
    types: [opened, synchronize]
  schedule:
    - cron: "0 2 * * *"           # 02:00 UTC daily
  workflow_dispatch:              # manual "Run workflow" button
    inputs:
      env: { type: choice, options: [dev, prod] }
```

---

## Matrix builds (test across versions)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: ${{ matrix.node }} }
      - run: npm ci && npm test
```

---

## Secrets & variables

```yaml
steps:
  - run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}          # repo/org/environment secret
      REGION:  ${{ vars.AWS_REGION }}          # non-secret variable
```

Set with the CLI:

```bash
gh secret set API_KEY --body "sk-..."
gh variable set AWS_REGION --body "ap-southeast-2"
```

> Secrets are masked in logs and never exposed to workflows triggered from forks. Never `echo` a secret.

---

## Advanced

### Job dependencies & outputs

`needs` orders jobs; pass data via `outputs`.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.version }}
    steps:
      - id: v
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build                                   # waits for build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"
```

### OIDC — deploy to AWS without static keys

Exchange a short-lived GitHub token for AWS creds via IAM role trust — no long-lived secrets in the repo. The recommended way to authenticate to cloud.

```yaml
permissions:
  id-token: write          # required for OIDC
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: ap-southeast-2
      - run: aws s3 sync ./dist s3://my-bucket
```

> **Grounded:** GitHub and AWS both [recommend OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) over storing `AWS_ACCESS_KEY_ID` secrets — credentials are short-lived and scoped per workflow.

### Caching dependencies

```yaml
- uses: actions/setup-node@v4
  with: { node-version: 20, cache: npm }     # built-in cache for npm/yarn/pnpm

# or generic cache for anything
- uses: actions/cache@v4
  with:
    path: ~/.cache/my-tool
    key: ${{ runner.os }}-${{ hashFiles('lockfile') }}
```

### Reusable workflows

Call one workflow from another (DRY across repos):

```yaml
# caller
jobs:
  ci:
    uses: my-org/.github/.github/workflows/node-ci.yml@main
    with: { node-version: 20 }
    secrets: inherit
```

### Concurrency (cancel superseded runs)

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true        # new push cancels the running build on that branch
```

### Permissions (least privilege)

```yaml
permissions:
  contents: read                  # default to read-only; grant only what a job needs
```

### Environments (gated deploys)

```yaml
jobs:
  deploy:
    environment: production        # can require manual approval / restrict secrets
    runs-on: ubuntu-latest
```

---

## Practical recipes

**Run only on PRs that touch code (skip docs):**
```yaml
on:
  pull_request:
    paths: ["src/**", "package.json"]
```

**Run a job only on tag pushes (release):**
```yaml
if: startsWith(github.ref, 'refs/tags/v')
```

**Post test results / fail the PR on lint:**
```yaml
- run: npm run lint        # non-zero exit fails the job → blocks the PR check
```

**Trigger a workflow manually with inputs:**
```bash
gh workflow run deploy.yml -f env=prod
```

**Watch a run from the terminal:**
```bash
gh run watch
gh run view --log-failed       # just the failed step logs
```

**Re-run only failed jobs:**
```bash
gh run rerun <run-id> --failed
```

---

## Tips

- Pin actions to a major version (`actions/checkout@v4`); pin to a full SHA for security-sensitive ones.
- Use **OIDC** for cloud auth — stop storing long-lived `AWS_*` keys as secrets.
- Set `permissions: contents: read` at the top and widen per job; the default token is broad.
- Add `concurrency` with `cancel-in-progress` so pushes don't pile up redundant runs.
- Cache dependencies (`cache: npm`) — biggest easy win on CI time.
- `fail-fast: false` in a matrix when you want every combo's result, not just the first failure.
- Keep secrets out of `run:` echoes and forked-PR workflows; they're masked but don't tempt fate.
