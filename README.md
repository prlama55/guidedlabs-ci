# guidedlabs-ci

One CI flow for every Guided Labs product. Each product's `.github/workflows/ci.yml` calls the reusable workflow
`.github/workflows/node-ci.yml` from this repository and only says *what* each job runs.

```
pull request into develop / stage / main, push to stage / main
          │
          ├─ gate ─────────── "already passed on stage?" (push to main only)
          │
          ├─ quick ────────── install → dependency audit → lint / typecheck / unit tests / build
          ├─ database ─────── fresh Postgres (all migrations) → pgTAP / RLS checks        ┐ skipped when the
          ├─ integration ──── local Supabase (Auth, REST, Storage) → tests against it     │ commit already
          └─ e2e (1..N) ───── optional Supabase → build → Playwright, sharded             ┘ passed on stage
          │
          └─ product jobs ─── (in the product's caller: migrations, desktop builds, ...)  needs: ci
```

Every product also has the same hardening: `permissions: contents: read` by default, actions pinned to a commit,
Dependabot for npm and GitHub Actions (weekly, into `develop`), and `concurrency` that never cancels a run on
`stage` or `main`. Deploy workflows are unchanged and stay in each product.

## Setup (once)

1. Create the GitHub repository `prlama55/guidedlabs-ci` and push this folder:
   ```bash
   git remote add origin git@github.com:prlama55/guidedlabs-ci.git
   git push -u origin main
   git tag v1 && git push origin v1
   ```
2. If the repository is private: **Settings → Actions → General → Access →** "Accessible from repositories owned
   by the user 'prlama55'". Without this, product runs fail with "workflow was not found".
3. Merge each product's CI branch. Products call `prlama55/guidedlabs-ci/.github/workflows/node-ci.yml@v1`.

## Changing the flow

Change `node-ci.yml` on a branch, test it from one product by pointing its caller at the branch
(`@my-branch`), then merge and move the tag:

```bash
git tag -f v1 && git push -f origin v1     # compatible change: every product picks it up
git tag v2 && git push origin v2           # breaking change (inputs renamed/removed): bump products one by one
```

## Inputs

| Input | Default | What it does |
| --- | --- | --- |
| `package-manager` | `pnpm` | `pnpm` or `npm` (must match the lockfile) |
| `node-version` | `24` | |
| `audit-level` | `moderate` | `pnpm/npm audit --audit-level`; `off` skips it |
| `audit-blocking` | `true` | `false` turns a failed audit into a warning (while a product's known vulnerabilities are fixed) |
| `quick` | required | script for the quick job |
| `quick-postgres` | `false` | plain Postgres (`pgvector/pgvector:pg16`, postgres/postgres, port 5432) for the quick job |
| `quick-timeout` | `20` | minutes |
| `database` | — | script for the database job (Supabase with only Postgres; `DB_URL` set) |
| `integration` | — | script for the integration job (`API_URL`, `ANON_KEY`, `SERVICE_ROLE_KEY`, `DB_URL` set) |
| `e2e` | — | end-to-end script (`SHARD`, `SHARDS` set, plus the Supabase variables when it runs) |
| `e2e-supabase` | `true` | start a local Supabase (with Mailpit) for e2e |
| `e2e-shards` | `1` | machines the e2e job is split over |
| `e2e-timeout` | `30` | minutes |
| `e2e-report-path` | `playwright-report` | uploaded when a shard fails |
| `next-cache` | `false` | cache `.next/cache` for the e2e build |
| `supabase-cli-version` | — | install this CLI; empty = the project's `supabase` dev dependency |
| `skip-heavy-when-tested-on-stage` | `true` | on a push to main, skip database/integration/e2e when the same commit passed CI on stage |

Scripts run in bash with `set -euo pipefail`. Empty `database`, `integration` or `e2e` means the job is skipped.

## Caller template

```yaml
name: CI # deploy workflows may listen for "CI" with workflow_run: keep the name

on:
  push:
    branches: [stage, main]
  pull_request:
    branches: [develop, stage, main]
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/stage' && github.ref != 'refs/heads/main' }}

permissions:
  contents: read

jobs:
  ci:
    uses: prlama55/guidedlabs-ci/.github/workflows/node-ci.yml@v1
    permissions:
      contents: read
      actions: read # the stage check reads workflow runs
    with:
      quick: |
        pnpm lint
        pnpm typecheck
        pnpm test
        pnpm build
```
