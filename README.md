# codyverge/.github — fleet CI spine

Org-default **reusable workflows** so the fleet's repos share one maintained CI
definition instead of N drifting copies (Phase 2 of the CI/CD engagement).

## Available reusable workflows

| Workflow | For | Key inputs (defaults) |
|---|---|---|
| `node-ci.yml` | Node/TS (vitest, node:test) | `node-version` (22), `test-script` (`test:ci`), `run-typecheck` (true), `run-build` (false) |
| `python-ci.yml` | Python (pytest) | `python-version` (3.11), `install-command`, `test-command` (`bash ci/test-ci.sh`) |
| `go-ci.yml` | Go (e.g. alena-tui) | `go-version` (1.23) |

## How to adopt (replace a repo's inline `ci.yml`)

```yaml
# .github/workflows/ci.yml in the consuming repo
name: ci
on:
  pull_request:
  push:
    branches: [main, master]
concurrency:                      # caller owns triggers + concurrency
  group: ci-${{ github.workflow }}-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
permissions:                      # least-privilege: cap the caller token
  contents: read
jobs:
  ci:
    uses: codyverge/.github/.github/workflows/node-ci.yml@main
    with:
      node-version: '22'
```

That's the whole consuming workflow. Pin `@main` for always-latest, or a tag/SHA to freeze.

## The contract: the skip-gate stays in the repo

These workflows run the repo's **own** `test:ci` / `test-command`, which MUST
self-enforce **`passed>0 AND skipped==0`** (a green run that silently skipped the
work is green theater). The per-runner gate lives in each repo as
`ci/test-ci.mjs` (node) or `ci/test-ci.sh` (python) — it decides which suites are
hermetic. These workflows own only the boilerplate (checkout, toolchain, deps,
PIN-5 dead-port env so the hermetic tier can't reach a real/box-local DB).

- **node:test TAP** prints `# skipped N` (NOT `# skip N`) — match the right token.
- Tier-2 (real-DB integration) is a separate per-repo `ci-db.yml` on a nightly
  trigger (`supabase start` / a `pgvector` service container) — not these.

## Access

This is a **public** repo so that repos in *any* org can call these workflows —
GitHub blocks cross-org calls to *private* reusable workflows, and the fleet
spans the `codyverge` and `marchonai` orgs. Public exposes only generic CI
boilerplate (no secrets — those always come from the caller's secret store at
runtime), keeping one maintained pipeline definition instead of per-org copies.

## Conventions baked in
`actions/checkout@v5` · `actions/setup-node@v6` / `setup-python@v5` / `setup-go@v5`
· node 22 · `npm ci` · `--if-present` tolerance · `permissions: contents: read`
· inputs passed via `env:` (never interpolated into a shell — injection-safe).
