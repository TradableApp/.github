# Tradable CI/CD Workflow Conventions

Canonical house style for GitHub Actions across all `TradableApp` repositories. New repos should start from the **org workflow templates** (Actions → New workflow → "Tradable CI (Bun)" / "Tradable CodeQL"); existing repos should match this document. The templates live in [`workflow-templates/`](./workflow-templates) in this repo.

> Why this exists: uniform workflows let us require **one** status-check name (`CI`) org-wide in branch rulesets, keep supply-chain risk down via SHA pinning, and make every repo's CI legible at a glance.

## Core rules

| Rule | Value |
|------|-------|
| CI job id **and** name | `CI` (so the required status check is `CI` everywhere) |
| CodeQL job name | `Analyze` — **no `strategy.matrix`** (matrix renames the check to `Analyze (javascript-typescript)`) |
| Runtime | **Bun**, pinned: `bun-version: 1.3.14` |
| Install | `bun install --frozen-lockfile` (commit `bun.lock`; no `package-lock.json`/`yarn.lock`) |
| Action pinning | Pin every `uses:` to a full **commit SHA** + trailing `# vX.Y.Z` comment. Never a bare tag. |
| Least privilege | CI jobs declare `permissions: contents: read`. CodeQL declares `actions: read` + `contents: read` + `security-events: write`. |
| Concurrency | Every workflow has a `concurrency` block keyed on `${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true`. |
| Triggers | `push` + `pull_request` on the default branch; CodeQL adds `schedule: cron '0 0 * * 0'`. |

## Step / phase naming

Steps that run commands are **named**; self-evident `uses:` setup steps (checkout, setup-bun) may stay unnamed in CI. Names are action-first, **no `Run ` prefix**:

- Single-target: `Install dependencies`, `Lint`, `Build`, `Test`, `Coverage`, `Typecheck`, `Compile contracts`.
- Multi-package repos disambiguate with a parenthetical target — and qualify **all** instances:
  - `Test (contracts)` / `Test (oracle)` (tokenized-ai-agent)
  - `Test (root)` / `Test (plugin-senseai)` / `Test (plugin-telegram-senseai)` (sense-ai-core)
  - `Install root dependencies` / `Install oracle dependencies`
- Do **not** mix `Test` and `Run tests` and `Test Step` — pick `Test` (+ qualifier).

## CodeQL specifics

- One matrix-free `analyze` job → uniform `Analyze` check name.
- Always set `category: '/language:javascript-typescript'` on the analyze step.
- CodeQL JS/TS needs **no build and no dependency install** — don't add `setup-bun`/`bun install` to the CodeQL workflow.
- **Private-repo gate:** keep `if: ${{ !github.event.repository.private }}` on the `analyze` job. Code scanning is free on public repos but a paid per-committer feature on private ones; the gate skips it (neutral, $0) on private repos and auto-activates if a repo goes public.

## Pinned action SHAs (current)

Bump via Dependabot (`github-actions` ecosystem, enabled in every repo). Keep the version comment in sync.

| Action | SHA | Version |
|--------|-----|---------|
| `actions/checkout` | `de0fac2e4500dabe0009e67214ff5f5447ce83dd` | v6.0.2 |
| `oven-sh/setup-bun` | `0c5077e51419868618aeaa5fe8019c62421857d6` | v2.2.0 |
| `github/codeql-action/*` | `03e4368ac7daa2bd82b3e85262f3bf87ee112f57` | v3 |

## Private cross-repo submodules

Repos consuming the private `sense-ai-shared-schema` submodule (`tokenized-ai-agent`, `sense-ai-core`) can't fetch it with the default `GITHUB_TOKEN`. Resolve the pinned submodule SHA, then `actions/checkout` it explicitly into `packages/shared-schema`. The token is being migrated from the org `SUBMODULE_PAT` to per-repo read-only **deploy keys** (least privilege) — see the SenseAI E2E plan's hardening section.

## Per-project variation (allowed)

CI **phases** differ by project type and that's fine — Solidity repos compile + coverage, the subgraph runs codegen + build, the dApp builds per Vite mode. What must stay uniform is the **structure**: job name, permissions, concurrency, pinned SHAs, Bun version, and the step-naming convention above.

## Dependabot

Every repo ships `.github/dependabot.yml` with the `github-actions` ecosystem (keeps pinned SHAs current) plus the `bun` ecosystem covering **every** directory that has its own `bun.lock`. Conventions:

- **Unquoted scalars** (`package-ecosystem: bun`, `directory: /`, `interval: weekly`) — no mixed single/double quoting.
- **Multiple Bun lockfile dirs** use one entry with a `directories:` list (not repeated `directory:` entries).
- **Commit message:** `commit-message: { prefix: chore, include: scope }` on every ecosystem → renders as `chore(deps): …`. Do **not** put the scope in the prefix (`prefix: chore(deps)` + `include: scope` doubles it to `chore(deps)(deps): …`).

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    commit-message:
      prefix: chore
      include: scope

  - package-ecosystem: bun
    directories:
      - /
      - /path/to/nested/package
    schedule:
      interval: weekly
    commit-message:
      prefix: chore
      include: scope
```
