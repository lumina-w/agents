# agents

Central repo of reusable GitHub Actions workflows for the `lumina-w`
organization. Every workflow here declares only `on: workflow_call`, with one
exception (`repo-sync.yml`, [below](#repo-syncyml)): it does nothing by
itself. Each product repo (and each `<project>-docs` repo) keeps a thin
caller that sets the triggers and calls the workflow pinned to the major tag,
so the logic lives once, here.

- **Current version:** `v4`, which points to `v4.7.0` (commit `65812d6`,
  agents#21). See [Versioning](#versioning).
- **Visibility:** private. Only repos of the organization can call these
  workflows.
- **Docs in this repo:**
  - `README.md` (this file): what each workflow does, who uses it, how to call it.
  - [`AGENTS.md`](AGENTS.md): the dev agent in detail (modes, layers, guardrails, checkpoint).
  - [`CONTRIBUTING.md`](CONTRIBUTING.md): how to change a workflow, test it and release it.

What does not live here: the git conventions (commit format, scopes,
`dev -> stg -> main` rules) belong to
[`@lumina-w/dev-standards`](https://github.com/lumina-w/dev-standards), and
the personal Claude Code configuration belongs to `wavival-coding-config`.

## Catalog

| Workflow | Purpose | Status |
|---|---|---|
| `dev-agent.yml` | Claude dev agent: implements labeled issues, iterates on `@claude` comments, fixes its own PRs when CI fails. Opens PRs against `dev`, never merges. | In use by the 8 product repos |
| `automerge-dev.yml` | Arms squash auto-merge on every non-draft PR into `dev` (not forks, not Dependabot). | In use by the 8 product repos |
| `delete-merged-branches.yml` | Deletes remote branches whose PR was merged into `dev`, when the branch tip is still the PR head. | In use by the 8 product repos |
| `docs-sync.yml` | Daily sync of a `<project>-docs` repo from the `dev` branch of its source repos. | In use by the 3 docs repos |
| `shared-commitlint.yml` | Conventional Commits lint of the commits each push introduces. | In use by the 8 product repos |
| `shared-pr-title.yml` | Conventional Commits lint of the PR title (the squash commit). | In use by the 8 product repos |
| `shared-validate-pr-base.yml` | Enforces the PR direction `dev -> stg -> main`. | In use by the 8 product repos |
| `shared-promote-dev-to-stg.yml` | Opens or reuses the `dev -> stg` promotion PR, waits for its required checks and merges it. | In use by the 8 product repos |
| `shared-ci-vite-react.yml` | Lint, format check, tests and production build for a Vite + React frontend, plus a build-only Docker check. | New, replaces terracore-front's ci.yml `lint-build-test` and `docker-build` jobs (not yet adopted) |
| `shared-claude-code-review.yml` | Claude code review with inline comments on the PR. | In use by 5 repos |
| `shared-codeql.yml` | CodeQL analysis for one language, informational by default. | In use by 3 repos |
| `shared-security-audit-node.yml` | `npm audit` or `pnpm audit`, informational by default. | In use by 3 repos |
| `shared-security-audit-python.yml` | `pip-audit` (blocking by default) and `bandit` (informational by default). | In use by 1 repo |
| `shared-ci-django.yml` | Django backend CI: ruff/black lint, pytest against a throwaway PostgreSQL, migration check, Docker build. | New, not yet called by any repo |
| `shared-gitleaks.yml` | Secret scan of the PR diff or the pushed commits. | In use by 2 repos |
| `shared-cd-docker-publish.yml` | Builds a Docker image, scans it with Trivy, pushes it to GHCR and optionally sends a `repository_dispatch`. | In use by 2 repos |
| `shared-ci-astro.yml` | Astro/pnpm CI: format/lint/typecheck/unit tests, build with a dist size warning, Playwright E2E. | New, not yet called by any repo |
| `shared-lighthouse.yml` | Lighthouse CI (performance, accessibility, SEO) for pnpm projects. | New, not yet called by any repo |
| `shared-release.yml` | Creates the tag and the GitHub release of one version, after checking the version, the version file and the changelog section. Implements the release standard of dev-standards. | New, not yet called by any repo |

| Action | Purpose | Status |
|---|---|---|
| `.github/actions/start-postgres` | Starts a throwaway PostgreSQL server inside the job, replacing `services: postgres` on runners without a Docker daemon. | New, for terracore-back's test jobs |

The ten `shared-*.yml` workflows were added by agents#13 (released as
`v4.4.0`) and extended by agents#14 (runner label arrays and repo commitlint
dependencies, `v4.5.0`). Both PRs are merged.

## Who calls what

Verified on 2026-09-25 by reading the `uses: lumina-w/agents/...@v4` lines on
the `dev`, `stg` and `main` branches of every repo in the organization. `dev`
is the default branch of all product repos, so scheduled callers run from it.

| Workflow | Callers (on `dev` and `stg`) |
|---|---|
| `dev-agent.yml` | terracore-back, terracore-front, terracore-page, okroot-back, okroot-front, okroot-page, blog-w, luminaw-page (`claude.yml`) |
| `automerge-dev.yml` | the same 8 (`automerge-dev.yml`) |
| `delete-merged-branches.yml` | the same 8 (`delete-merged-branches.yml`, `30 4,16 * * *`) |
| `shared-commitlint.yml` | the same 8 (`commit-lint.yml`); terracore-back with `install-dependencies: true` |
| `shared-pr-title.yml` | the same 8 (`pr-title.yml`); terracore-back with `install-dependencies: true` |
| `shared-validate-pr-base.yml` | the same 8 (`validate-pr-base.yml`) |
| `shared-promote-dev-to-stg.yml` | the same 8 (`promote-dev-to-stg.yml`, `0 10 * * *`) |
| `shared-claude-code-review.yml` | okroot-back, okroot-front, okroot-page, blog-w, luminaw-page (`claude-code-review.yml`) |
| `shared-codeql.yml` | terracore-back, terracore-front (`codeql.yml`, Mondays 06:00 UTC plus push/PR), terracore-page (`codeql.yml`, `workflow_dispatch` only) |
| `shared-security-audit-node.yml` | terracore-front (`security.yml`), terracore-page (`ci.yml`), okroot-page (`ci.yml`) |
| `shared-security-audit-python.yml` | terracore-back (`security.yml`) |
| `shared-gitleaks.yml` | terracore-back (`ci.yml`), terracore-front (`ci.yml`, on the self-hosted runner `["self-hosted", "build", "terracore-front"]`) |
| `shared-cd-docker-publish.yml` | okroot-back (`cd-staging.yml`, `cd-production.yml`), okroot-front (`docker-build-push.yml`) |
| `docs-sync.yml` | terracore-docs, okroot-docs, luminaw-docs (`docs-daily-sync.yml`, `0 11 * * *`, 06:00 in Bogotá) |
| `shared-release.yml` | none yet (`release.yml`, `workflow_dispatch`; blog-w is the first to adopt it) |

Exceptions to `@v4` and to `ubuntu-latest` in the table above:

- terracore-page's `commit-lint.yml` calls
  `shared-commitlint.yml@feat/self-hosted-runner-support` with
  `runner-label: '["self-hosted", "build", "lumina-w"]'` on `dev`, `stg` and
  `main`. It is the test caller of agents#17, merged as `dffd129`. It breaks
  if that branch is deleted before the caller goes back to `@v4`.
- terracore-back passes `runner-label: '["self-hosted", "build", "lumina-w"]'`
  in `ci.yml` (gitleaks), `codeql.yml`, `pr-title.yml`, `security.yml` and
  `validate-pr-base.yml`, on `dev` only (terracore-back#189).

State of `main` in the product repos (it only changes with each
`stg -> main` promotion):

- terracore-back, terracore-front, terracore-page: same callers as `dev`
  (10, 10 and 9). terracore-back's `main` caught up with the
  `stg -> main` promotion of 2026-09-25.
- okroot-back, okroot-front, okroot-page, blog-w, luminaw-page: no callers
  yet; `main` still has the per-repo workflows from before the migration.

Logic that still lives as a per-repo copy although a shared workflow exists:
the Docker publish job of terracore-front (`cd-staging.yml`,
`cd-production.yml`).

`wavival/wavival.dev` is in the personal account, not in the organization,
and does not call any of these workflows.

## Calling a workflow

The caller lives in the consumer repo's `.github/workflows/`, sets the
triggers and pins the major tag:

```yaml
name: Commit lint
on:
  push:
    branches: ['**']
permissions:
  contents: read
jobs:
  commitlint:
    uses: lumina-w/agents/.github/workflows/shared-commitlint.yml@v4
    with:
      runner-label: ubuntu-latest
```

Rules that apply to every workflow:

- **Pin `@v4`,** never `@main` or a branch. `@v4.Y.Z` pins one exact release.
- **Triggers belong to the caller.** The workflows only declare
  `workflow_call`.
- **Permissions:** each job here declares its minimum permissions, and the
  caller job must grant at least those (a reusable workflow cannot get more
  than its caller). Exception: `shared-cd-docker-publish.yml` uses the
  caller's.
- **Secrets:** pass them one by one under `secrets:`, or `secrets: inherit`
  (what `dev-agent.yml` and `docs-sync.yml` callers do). Secrets are not
  available to workflows triggered by Dependabot unless they also exist in the
  Dependabot secret store.
- **Check names** show up as `<caller job> / <job name>`. Moving a repo to a
  shared workflow changes the names, so the required checks of its branch
  protection must be updated in the same change.

### Runner selection (`runner-label`)

Every `shared-*.yml`, `dev-agent.yml`, `automerge-dev.yml` and
`delete-merged-branches.yml` accept `runner-label`:

| Value | Runs on |
|---|---|
| empty (default) | `ubuntu-latest` |
| a label, e.g. `terracore-vps` | the self-hosted runner with that label |
| a JSON array, e.g. `'["self-hosted", "build", "lumina-w"]'` | a runner that has all those labels |

`docs-sync.yml` has no runner input: it always runs on `ubuntu-latest`.

The organization's ephemeral runners carry `["self-hosted", "build", "lumina-w"]`:
one single-use runner per job in a rootless Docker container on the build VPS,
with no Docker daemon inside the job, no `sudo` and one job at a time for the
whole organization. A job can only use them if it needs no `services:`,
`container:`, container actions (`runs: using: docker`) or `apt-get`, and
only the tools the runner image ships.

### Secrets by workflow

| Workflow | Secret | Required | Use |
|---|---|---|---|
| `dev-agent.yml` | `CLAUDE_CODE_OAUTH_TOKEN` | yes | Claude authentication |
| | `DOCS_SOURCES_READ_TOKEN` | no | Read-only org token to clone `<project>-docs` as context |
| | `PROMOTE_TOKEN` | no | Commits the checkpoint and marks the PR ready so PR workflows run again |
| `automerge-dev.yml` | `PROMOTE_TOKEN` | yes | Arms auto-merge; the resulting push to `dev` triggers the repo's workflows |
| `delete-merged-branches.yml` | none | | Uses the workflow token (`contents: write`) |
| `docs-sync.yml` | `CLAUDE_CODE_OAUTH_TOKEN` | yes | Claude authentication |
| | `DOCS_SOURCES_READ_TOKEN` | yes | Reads the source repos |
| `shared-commitlint.yml` | `DEV_STANDARDS_DEPLOY_KEY` | no | Installs `@lumina-w/dev-standards` over `git+ssh`, with `install-dependencies: true` or a non-empty `dev-standards-ref` |
| `shared-pr-title.yml` | `DEV_STANDARDS_DEPLOY_KEY` | no | Same |
| `shared-validate-pr-base.yml` | none | | |
| `shared-promote-dev-to-stg.yml` | `PROMOTE_TOKEN` | yes | A PR opened with the workflow token would not trigger the required checks |
| `shared-claude-code-review.yml` | `CLAUDE_CODE_OAUTH_TOKEN` | yes | Claude authentication |
| `shared-codeql.yml` | none | | |
| `shared-security-audit-node.yml` | none | | |
| `shared-security-audit-python.yml` | none | | |
| `shared-gitleaks.yml` | `GITLEAKS_LICENSE` | no in the schema, needed in practice | The action requires it for organization repos. It must also exist in the Dependabot secret store |
| `shared-cd-docker-publish.yml` | `BUILD_ARGS` | no | Secret build args, `KEY=value` per line |
| | `DISPATCH_TOKEN` | no (yes when dispatching to another repo) | Token for the `repository_dispatch` |
| `shared-ci-astro.yml` | none | | |
| `shared-lighthouse.yml` | none | | |
| `shared-release.yml` | none | | Uses the workflow token (`contents: write`) to create the tag and the release |
| `repo-sync.yml` | `BRANCH_PROTECTION_TOKEN` | yes | Reads and writes branch protection and repo settings on every managed repo, and reads `rules/flow.json` from dev-standards |

`CLAUDE_CODE_OAUTH_TOKEN`, `DOCS_SOURCES_READ_TOKEN` and `PROMOTE_TOKEN` are
organization secrets. `DOCS_SOURCES_READ_TOKEN` is a fine-grained PAT with
Contents read-only on the code repos, `agents` and the three `*-docs` repos.
`BRANCH_PROTECTION_TOKEN` is also an organization secret: a fine-grained PAT
with Administration: write and Contents: read on the 8 product repos, plus
Contents: read on `dev-standards` (to read `rules/flow.json`). Nobody has
created it yet; `repo-sync.yml` fails its first step until it exists.

<!-- dev-agent:start -->
## Dev agent (`dev-agent.yml`)

Summary here; the full reference (modes, trigger rules, the nine layers,
guardrails, checkpoint, labels) is in [`AGENTS.md`](AGENTS.md).

The agent works from a labeled issue to a PR against `dev`, and stops there.
The rest of the flow belongs to the repo's workflows:

```
issue + label --> dev agent --> PR into dev --> CI --(green)--> automerge-dev --> squash merge into dev
                                   ^             |                                       |
                                   |           (red)                                     v
                                   +-- fix mode (max 2) <--+          delete-merged-branches (every 12 h)
                                                                                         |
                                            promote-dev-to-stg (10:00 UTC) --> stg --(person)--> main
```

| Event | Mode | Default model | Turns |
|---|---|---|---|
| Label `dev-ft-ready` on an issue | implement | `opus` | 80 |
| Label `dev-ft-small` on an issue | implement | `sonnet` | 40 |
| Comment containing `@claude` on a PR | iterate | `sonnet` | 40 |
| The `CI` workflow fails on a PR labeled `dev-agent` | fix | `opus` | 80 |

### Minimal caller (`.github/workflows/claude.yml`)

```yaml
name: Claude Code
on:
  issues:
    types: [labeled]
  issue_comment:
    types: [created]
  workflow_run:
    workflows: [CI]
    types: [completed]
jobs:
  claude:
    if: >-
      (github.event_name == 'issues' && (github.event.label.name == 'dev-ft-ready' || github.event.label.name == 'dev-ft-small')) ||
      (github.event_name == 'issue_comment' && github.event.issue.pull_request && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'workflow_run' && github.event.workflow_run.event == 'pull_request' && github.event.workflow_run.conclusion == 'failure')
    uses: lumina-w/agents/.github/workflows/dev-agent.yml@v4
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    secrets: inherit
    with:
      docs_repo: terracore-docs
      toolchain: npm
      node_version: "20.x"
      install_command: npm ci
      verify_commands: |
        npm run lint
        npm run test
        npm run build
```

Fix mode needs both a workflow named `CI` and the `workflow_run` trigger in
the caller. luminaw-page has a `CI` workflow (`ci.yml` on `dev` and `stg`, not
yet on `main`), but its `claude.yml` only listens to `issues` and
`issue_comment`, so it still has no fix mode. The other seven callers have
both.

### Inputs

| Input | Required | Default | Use |
|---|---|---|---|
| `toolchain` | yes | | `python`, `npm`, `pnpm` or `none` |
| `python_version` | | | Python version of the repo |
| `python_cache_path` | | | Requirements file used as the pip cache key |
| `node_version` / `node_version_file` | | | Node version, or a file such as `.nvmrc` (used when `node_version` is empty) |
| `install_command` | | | Installs dependencies before the agent starts (`npm ci`, `pnpm install --frozen-lockfile`, `pip install -r ...`) |
| `env_vars` | | | `KEY=value` per line for local checks. Never secrets |
| `verify_commands` | yes | | The repo's done-criteria commands from its `CLAUDE.md`, one per line. The only checks the agent may run |
| `extra_allowed_tools` | | | Extra Claude Code permission rules, one per line (`Bash(npm run format)`) |
| `context_files` | | | Repo docs to read besides `CLAUDE.md` and `AGENTS.md` |
| `docs_repo` | | | The project's docs repo (`terracore-docs`, `okroot-docs`, `luminaw-docs`), read-only context |
| `checkpoint_file` | | `.claude/CHECKPOINT.md` | Checkpoint the workflow commits to each PR |
| `base_branch` | | `dev` | Branch the agent starts from and opens PRs against |
| `model` / `small_model` | | `opus` / `sonnet` | Model for full runs, and for `dev-ft-small` and iterate |
| `max_turns` / `small_max_turns` | | `80` / `40` | Turn limit per size |
| `max_fix_rounds` | | `2` | Maximum fix rounds per PR |
| `timeout_minutes` | | `30` | Job time limit |
| `runner-label` | | `''` | Runner label, see [Runner selection](#runner-selection-runner-label). Empty runs on `ubuntu-latest` |

### Callers today

| Repo | Toolchain | Verification (`verify_commands`) | Docs repo |
|---|---|---|---|
| terracore-back | Python 3.14 | ruff, black, `make test`, `make check`, `makemigrations --check` | terracore-docs |
| terracore-front | npm, Node 20.x | format:check, lint, type-check, test, build | terracore-docs |
| terracore-page | pnpm, `.nvmrc` | format:check, lint, typecheck, test, build | terracore-docs |
| okroot-back | Python 3.12 | `make lint`, `ruff format --check`, `make test`, `makemigrations --check` | okroot-docs |
| okroot-front | pnpm, Node 20 | typecheck, lint, test, build | okroot-docs |
| okroot-page | npm, Node 22 | lint, build | okroot-docs |
| blog-w | npm, Node 20 | lint, build | luminaw-docs |
| luminaw-page | npm, `.nvmrc` | format:check, build | luminaw-docs |

### Trying it

1. Open a small issue in a repo that calls it and add the `dev-ft-small` label.
2. Follow the run in the repo's Actions tab (job `claude / agent`).
3. The PR must arrive with the `dev-agent` label, the updated checkpoint and
   only the checks it actually ran marked as done.
<!-- dev-agent:end -->

## Other workflows

### `automerge-dev.yml`

Arms squash auto-merge on PRs into `dev`. Skips drafts, PRs from forks and
PRs opened by `dependabot[bot]` (those wait for a person). It only arms the
auto-merge: branch protection holds the merge until the required checks pass,
so `dev` must be protected with required checks. Uses `PROMOTE_TOKEN`, not the
workflow token, so the push to `dev` triggers the repo's workflows. The dev
agent's PRs are drafts until the workflow marks them ready, so the caller must
also trigger on `ready_for_review`. Input: `runner-label` (`''`).

### `delete-merged-branches.yml`

Deletes remote branches whose PR was merged into `base_branch` (`dev` by
default), only when the branch tip is still the PR head (a reused branch name
with new commits is kept). Never deletes `dev`, `stg`, `main`, the default
branch, protected branches or branches with an open PR. Inputs: `dry_run`
(`false`), `base_branch` (`dev`), `runner-label` (`''`). Callers schedule it every 12 hours because
"Automatically delete head branches" is not reliable with auto-merge.

### `repo-sync.yml`

The one workflow here that is not `on: workflow_call`: it runs directly in
this repo, triggered by a person through `workflow_dispatch`, and reaches
across the organization instead of being called by a single caller. It reads
`rules/flow.json` from `@lumina-w/dev-standards` at a pinned tag and, for
every repo listed in `.github/repo-sync.config.yml` (the 8 product repos),
applies its branch protection (required checks, the up-to-date requirement,
review count, deletion/force-push) and its repo-level merge settings
(`delete_branch_on_merge`, the allowed merge methods).

Detection, not assumption: for each repo and branch it reads that branch's
own `.github/workflows/*.yml` and checks which shared workflows
(`shared-ci-*.yml`, `shared-gitleaks.yml`, `shared-commitlint.yml`,
`shared-pr-title.yml`, `shared-validate-pr-base.yml`,
`shared-security-audit-*.yml`, `shared-codeql.yml`) it actually calls. Only
the required checks whose shared workflow is wired up there are set as
required; a branch with none wired up yet is left untouched, matching
`flow.json`'s own `required_checks_note`. `dev`, `stg` and `main` can be at
different points of the migration, so each branch is checked on its own, and
each looks up its own required-checks list (`dev`'s is a superset of `stg`'s
and `main`'s, since `dev-standards` 0.5.0).

Referencing a shared workflow is not enough for `analyze / analyze`: a caller
can gate its `codeql.yml` to `workflow_dispatch` only (no GitHub Advanced
Security on a private repo, see `shared-codeql.yml` below), where the
`analyze` job never runs on a pull request. Requiring that check there would
leave it permanently unresolved. So that marker also requires an active,
uncommented `pull_request:` trigger in the workflow text; a
`workflow_dispatch`-only caller like terracore-page's is correctly left out.

Before writing `required_status_checks`, it reads the branch's current
protection. A check already required there is kept only if `flow.json` has
never heard of it (a repo's own native CI, before it migrates to a
`shared-ci-*.yml`); a check `flow.json` does manage, but that this branch's
current list no longer requires, is dropped, since `flow.json` is the single
source for every context it can produce. This is what actually removes
`pr-title`, `commitlint` and `gitleaks` from `stg` and `main`'s required
checks once a repo has run this against `dev-standards` 0.5.0 or later.

`required_approving_review_count: 0` (every branch, today) is read as "do
not send `required_pull_request_reviews` at all", since the classic branch
protection endpoint may reject `0` there. `enforce_admins` is not in
`flow.json`; this always sends `false`.

Inputs: `dev_standards_ref` (`v0.6.0`), `repos` (comma-separated subset of the
config, `''` for all), `dry_run` (`true`). A dry run never calls a write API:
it only prints, per repo and branch, the exact protection payload it would
send, or why a branch was skipped, to the run summary. Review that output
before a real run, and scope `repos` to one repo for the first one.

<!-- docs-sync:start -->
### `docs-sync.yml`

Keeps each `lumina-w/<project>-docs` repo current from the `dev` branch of the
project's source repos. Each docs repo has a thin caller
(`docs-daily-sync.yml`) and a `docs-sync.config.yml` that lists the source
repos, the managed and protected docs and the weekday of the full audit.
Today the three callers run at the same time, `0 11 * * *` (06:00 in Bogotá),
and also accept `workflow_dispatch` with `force`.

| Case | What happens | Model |
|---|---|---|
| No new commits | Nothing runs | |
| Dependency bumps only (Dependabot, lockfiles) | No Claude. Script updates `changelog` (one row per repo with the count of bumps) and `checkpoint` | |
| Code commits | Script builds `changelog` (from `git log`) and `checkpoint` (from each repo's `.claude/CHECKPOINT.md`). Claude gets `_changes.md` with the diff and updates only the affected docs | `sonnet`, 70 turns |
| Full audit: `full_audit_weekday` (Sunday in all three configs), first sync of a repo, or `force: true` | Claude reviews every managed doc | `opus`, 150 turns |

- The `opus` and `sonnet` aliases resolve to the latest model that
  `CLAUDE_CODE_OAUTH_TOKEN` can use.
- The incremental limit was 40 turns until v4.6.0. The scheduled run of
  luminaw-docs on 2026-09-24 needed 53 and failed, so it is 70 now.
- Optional inputs `model` and `max_turns` override both modes. `auto_merge`
  (`true`) and `config_path` (`docs-sync.config.yml`) are also inputs.
- Report: `.docs-sync/last-report.md` is written by Claude, followed by a
  section written by the script. Each run starts without the previous report.
  If Claude runs but writes none, the script says so, instead of reporting
  "no changes that need Claude" next to the docs assigned to Claude. The sync
  of 2026-09-23 in luminaw-docs produced that contradiction before v4.6.0.
- Guards: only root `.md` files of the docs repo; Claude cannot touch
  `changelog`, `checkpoint` or protected docs; one current file per doc type.
- Publishing: a PR with auto-merge, or a direct push to the docs repo's
  `main` if the organization does not let Actions open PRs.
- No Google Drive mirror (removed on 2026-09-23): the docs live only in
  GitHub.
- luminaw-docs also reads this repo (`agents`, branch `main`) as a source.
<!-- docs-sync:end -->

<!-- shared-workflows:start -->
## Shared CI/CD workflows (`shared-*.yml`)

Each one replaces logic that used to be copied in every product repo. The
"Origin" line names the per-repo file it came from.

Third-party actions are pinned: `actions/checkout@v7`,
`actions/setup-node@v7`, `actions/setup-python@v7`, `pnpm/action-setup@v6.1.0`,
`docker/build-push-action@v7`, `docker/login-action@v4`,
`docker/metadata-action@v6`, `github/codeql-action@v4`,
`webfactory/ssh-agent@v0.10.0`,
`gitleaks/gitleaks-action@v3`, `anthropics/claude-code-action@v1`,
`peter-evans/repository-dispatch@v4`, `treosh/lighthouse-ci-action@v12`,
`actions/upload-artifact@v7`, `actions/github-script@v8` and `aquasecurity/trivy-action` by commit SHA
(v0.36.0).

### `shared-commitlint.yml`

Origin: `commit-lint.yml`. Lints the commits each push introduces
(`before..after`; a new branch, or a force push whose `before` is not an
ancestor, lints the commits no other branch contains) against the caller
repo's `.commitlintrc.json`. It runs `@commitlint/cli` directly instead of a
container action, so it also runs on the ephemeral self-hosted runners.

Commits authored by `dependabot[bot]` (git author name) are linted without the
header length rules the resolved config defines, the same exemption
`shared-pr-title.yml` gives Dependabot's PR titles. That covers the commits on
Dependabot's branches and the squash of its PRs, whose author is also
`dependabot[bot]`. The job writes `.commitlintrc.dependabot-commits.json` next
to the config, extending it and turning off only those rules; type, scope and
the rest still apply, so a title like `Bump the ... group` or a `deps-dev`
scope still fails. A push with no Dependabot commit is linted in one
commitlint run over the range; a push with one is linted commit by commit,
each against the config that applies to it. The git author name can be set by
anyone who pushes, so the exemption is only as strong as the header length
rule it relaxes: it never skips type, scope or any other rule.

- Caller trigger: `push: branches: ['**']`
- Permissions: `contents: read`
- Check: `Conventional Commits`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `config-file` | string | `.commitlintrc.json` | commitlint config of the repo. The job fails if the file is missing |
| `install-dependencies` | boolean | `false` | `true` runs `npm ci` first, for an `extends` that names a package from the repo's `package.json` (`@lumina-w/dev-standards`). Ties every job in the repo that runs `npm ci` to `DEV_STANDARDS_DEPLOY_KEY`, so it only fits a repo that already has a `package.json` for other reasons (terracore-back) |
| `dev-standards-ref` | string | `''` | A `dev-standards` tag (`v0.3.0`). Clones it straight into the scratch install so `extends: ["@lumina-w/dev-standards/commitlint"]` resolves with no change to the consumer's own `package.json`. Prefer this for a new consumer; ignored when `install-dependencies` is `true` |
| `node-version` | string | `'22'` | Node used to install and run commitlint |
| `commitlint-version` | string | `'21'` | Major version of `@commitlint/cli` (and `@commitlint/config-conventional` without `install-dependencies`) |

| Secret | Required | Use |
|---|---|---|
| `DEV_STANDARDS_DEPLOY_KEY` | no | Read-only deploy key to install `@lumina-w/dev-standards` over `git+ssh`. Used with `install-dependencies: true` or a non-empty `dev-standards-ref` |

### `shared-pr-title.yml`

Origin: `pr-title.yml`. PRs are squash-merged, so the title becomes the commit
on the base branch; it is linted with the same `.commitlintrc.json`. The title
is passed through the environment, never interpolated into the script.

PRs opened by `dependabot[bot]` (the PR author) are linted without the header
length rules the resolved config defines: `header-max-length`, and
`header-max-length-no-pr-suffix` in repos that extend `@lumina-w/dev-standards`.
Dependabot writes those titles and they often pass 72 characters (okroot-page's
group bumps reach 78). The job writes `.commitlintrc.dependabot-title.json` next
to the config, extending it and turning off only those rules, so type, scope and
the rest still apply. PRs opened by anyone else are linted as is.
`shared-commitlint.yml` gives Dependabot's commits the same exemption.

- Caller trigger: `pull_request: types: [opened, edited, synchronize, reopened]`
- Permissions: `contents: read`
- Check: `PR title (Conventional Commits)`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `config-file` | string | `.commitlintrc.json` | commitlint config of the repo |
| `node-version` | string | `'22'` | Node used to install commitlint |
| `commitlint-version` | string | `'21'` | Major version of `@commitlint/cli` and `@commitlint/config-conventional` |
| `install-dependencies` | boolean | `false` | `true` runs `npm ci` and lints from the repo root, for an `extends` that names a package of the repo. Only adds `@commitlint/cli`, without touching `package.json`. Ties every job in the repo that runs `npm ci` to `DEV_STANDARDS_DEPLOY_KEY` |
| `dev-standards-ref` | string | `''` | A `dev-standards` tag (`v0.3.0`). Clones it straight into the scratch install so `extends: ["@lumina-w/dev-standards/commitlint"]` resolves with no change to the consumer's own `package.json`. Prefer this for a new consumer; ignored when `install-dependencies` is `true` |

| Secret | Required | Use |
|---|---|---|
| `DEV_STANDARDS_DEPLOY_KEY` | no | Same as in `shared-commitlint.yml` |

### `shared-validate-pr-base.yml`

Origin: `validate-pr-base.yml`. A PR into `stg` must come from `dev`, a PR
into `main` must come from `stg`, and no other base is allowed. The exception
is a conflict-resolution branch whose tree matches the expected head exactly.

- Caller trigger: `pull_request: types: [opened, edited, synchronize, reopened]`
- Permissions: `contents: read`
- Check: `PR base must follow dev -> stg -> main`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |

### `shared-promote-dev-to-stg.yml`

Origin: `promote-dev-to-stg.yml`. Opens or reuses the promotion PR, waits for
its required checks and merges it. One promotion at a time per branch pair.

It merges with a merge commit by default, which keeps the source branch's
history on the target. Until v4.7.0 the default was `squash`, which dropped
that history: every later edit to lines a promotion had carried conflicted on
the next promotion PR, and GitHub runs no checks on a conflicting PR
(terracore-back#187, terracore-back#196). The repo must allow merge commits.

- Caller trigger: `schedule` (today `0 10 * * *`) and `workflow_dispatch`
- Permissions: `contents: read`, `pull-requests: write`, `checks: read`, `statuses: read`, `actions: read`
- Secrets: `PROMOTE_TOKEN` (required)

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `source-branch` | string | `dev` | Branch that is promoted |
| `target-branch` | string | `stg` | Target branch |
| `merge-method` | string | `merge` | `merge`, `squash` or `rebase` |
| `timeout-minutes` | number | `60` | Job time limit |

### `shared-ci-vite-react.yml`

Origin: the `lint-build-test` and `docker-build` jobs of terracore-front's
`ci.yml` (identical on `dev`, `stg` and `main`). `lint-build-test` installs
with `npm ci` and runs the format check, lint, tests and the production build
in that order. `docker-build` only builds the image, without pushing it, to
catch a broken Dockerfile before merge.

`docker-build` always runs on `ubuntu-latest` and ignores `runner-label`: the
organization's self-hosted runners are rootless containers with no Docker
daemon inside the job (see [Runner selection](#runner-selection-runner-label)),
so `docker/build-push-action` cannot run there at all.

- Caller trigger: the repo's CI (`pull_request`/`push`)
- Permissions: `contents: read`
- Checks: `Lint, build & test`, `Docker build (frontend)` and `gate` (green only when both of the above are, so branch protection can require one name regardless of stack)

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label for `lint-build-test` only |
| `node-version` | string | `"20.x"` | Node version for `lint-build-test` |
| `context` | string | `'.'` | Docker build context for `docker-build` |
| `dockerfile` | string | `''` | Dockerfile path from the repo root, for `docker-build`. Empty uses `<context>/Dockerfile` |
| `timeout-minutes-test` | number | `15` | Time limit of `lint-build-test` |
| `timeout-minutes-docker` | number | `15` | Time limit of `docker-build` |

```yaml
jobs:
  ci:
    uses: lumina-w/agents/.github/workflows/shared-ci-vite-react.yml@v4
    with:
      runner-label: '["self-hosted", "build", "lumina-w"]'
```

### `shared-claude-code-review.yml`

Origin: `claude-code-review.yml` (okroot-*, blog-w, luminaw-page). Runs the
`code-review` plugin, which posts inline comments on the PR. A newer push to
the same ref cancels the review in progress.

- Caller trigger: `pull_request: types: [opened, synchronize, ready_for_review, reopened]`
- Permissions: `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`
- Secrets: `CLAUDE_CODE_OAUTH_TOKEN` (required)

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `model` | string | `''` | Passed as `--model` (`claude-sonnet-5`, `sonnet`). Empty uses the action's default |
| `timeout-minutes` | number | `15` | Job time limit |

### `shared-codeql.yml`

Origin: `codeql.yml` (terracore-*). CodeQL for one language. What happens
when something fails depends on `blocking` and on the step:

- `Perform CodeQL Analysis` (analysis and SARIF upload) has
  `continue-on-error: ${{ !inputs.blocking }}`. With `blocking: false` (the
  default, terracore-back and terracore-front) a failure there, such as
  "Code Security must be enabled for this repository to use code scanning" on
  a private repo without it, leaves the `analyze` check green.
- With `blocking: true` the same failure turns the check red. terracore-page
  passes `blocking: true`. Its only run, by hand on 2026-09-25, failed in that
  step with that error.
- Checkout and `Initialize CodeQL` have no step-level `continue-on-error`. A
  failure there turns the `analyze` check red even with `blocking: false`;
  the job-level `continue-on-error` only keeps the workflow run itself from
  failing.

- Caller trigger: `push`/`pull_request` to `main`, `stg`, `dev` and `schedule`, or only `workflow_dispatch` in repos without GHAS (terracore-page)
- Permissions: `security-events: write`, `actions: read`, `contents: read`
- Check: `analyze`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `language` | string | required | `python`, `javascript-typescript`, ... |
| `queries` | string | `''` | Extra suite (`security-and-quality`). Empty uses the default suite |
| `blocking` | boolean | `false` | `true` fails the check when the analysis or the upload fails |
| `timeout-minutes` | number | `30` | Job time limit |

### `shared-security-audit-node.yml`

Origin: `security.yml` of terracore-front (npm) and the audit job of the
`ci.yml` of okroot-page (npm) and terracore-page (pnpm). Installs
dependencies and runs `npm audit` or `pnpm audit`.

- Caller trigger: whatever the repo uses (`push`/`pull_request`/`schedule`)
- Permissions: `contents: read`
- Check: `npm-audit` or `pnpm-audit`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `package-manager` | string | required | `npm` or `pnpm` |
| `node-version` | string | `''` | Node version (`20.x`, `22`) |
| `node-version-file` | string | `''` | Version file (`.nvmrc`) |
| `pnpm-version` | string | `''` | pnpm version. Empty reads `packageManager` from `package.json` |
| `working-directory` | string | `'.'` | Folder with `package.json` and the lockfile |
| `install-command` | string | `''` | Empty uses `npm ci` or `pnpm install --frozen-lockfile` |
| `audit-level` | string | `high` | `low`, `moderate`, `high` or `critical` |
| `blocking` | boolean | `false` | `true` fails the check on findings at or above `audit-level` |
| `timeout-minutes` | number | `15` | Job time limit |

### `shared-security-audit-python.yml`

Origin: `security.yml` of terracore-back. Two jobs: `pip-audit` (dependencies,
blocking by default) and `bandit` (SAST, informational by default).

- Caller trigger: whatever the repo uses (`push`/`pull_request`/`schedule`)
- Permissions: `contents: read`
- Checks: `pip-audit` and `bandit`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `python-version` | string | `''` | Python version (`3.14`, `3.12`) |
| `python-version-file` | string | `''` | Version file (`.python-version`) |
| `working-directory` | string | `'.'` | Folder where pip-audit runs |
| `requirements-file` | string | `requirements.txt` | Requirements audited by pip-audit, relative to `working-directory` |
| `pip-audit-version` | string | `2.9.0` | pip-audit version |
| `pip-audit-blocking` | boolean | `true` | `false` makes pip-audit informational |
| `bandit-version` | string | `1.9.4` | bandit version |
| `bandit-path` | string | `'.'` | Path bandit scans, relative to the repo root |
| `bandit-exclude` | string | `*/tests/*,*/migrations/*,*/.venv/*,*/venv/*` | Paths excluded from bandit |
| `bandit-blocking` | boolean | `false` | `true` fails the check on bandit findings |
| `timeout-minutes` | number | `15` | Time limit of each job |

### `shared-ci-django.yml`

Origin: the `lint-and-test` and `docker-build` jobs of `ci.yml`
(terracore-back). Two jobs: `lint-and-test` (ruff, black, Django checks,
pytest, against a throwaway PostgreSQL) and `docker-build` (`docker compose
build`, no push).

`lint-and-test` replaces the `services: postgres` container with
`./.github/actions/start-postgres`: service containers need a Docker daemon,
which the organization's ephemeral self-hosted runners do not have, so this
job can run on `runner-label` as well as on `ubuntu-latest`, with the same
database version and throwaway credentials as before. `docker-build` always
runs on `ubuntu-latest`: `docker compose build` itself needs a Docker
daemon, so it is never parameterized to `runner-label`.

ruff and black run from the repo root, where `pyproject.toml` lives, not
from `working-directory`. black is unpinned by default, matching
terracore-back today (the repo pins ruff but not black yet); set
`black-version` to pin it.

- Caller trigger: the repo's CI (`pull_request`)
- Permissions: `contents: read` (both jobs)
- Checks: `Lint & test (pytest)`, `Docker build (backend)` and `gate` (green only when both of the above are, so branch protection can require one name regardless of stack)

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner for `lint-and-test`. `docker-build` always runs on `ubuntu-latest` |
| `python-version` | string | `'3.14'` | Python version |
| `requirements-file` | string | `'backend/requirements/base.txt'` | Requirements file installed and cached, relative to the repo root |
| `working-directory` | string | `'backend'` | Django project directory (`manage.py`, `requirements/`), relative to the repo root |
| `ruff-version` | string | `'0.16.0'` | ruff version |
| `black-version` | string | `''` | Empty installs the latest black, unpinned (today's terracore-back behavior) |
| `compose-file` | string | `'docker-compose.prod.yml'` | Compose file built by `docker-build`, relative to `working-directory` |
| `compose-services` | string | `'web nginx'` | Space-separated services passed to `docker compose build` |
| `timeout-minutes-test` | number | `20` | Time limit of `lint-and-test` |
| `timeout-minutes-docker` | number | `15` | Time limit of `docker-build` |

### `shared-gitleaks.yml`

Origin: the `gitleaks` job of the `ci.yml` of terracore-back and
terracore-front. Scans the PR diff or the pushed commits for secrets.

- Caller trigger: the repo's CI (`pull_request`)
- Permissions: `contents: read`, `pull-requests: read`
- Secrets: `GITLEAKS_LICENSE`. The action requires it for organization repos; it has to exist in the Dependabot secret store too
- Check: `Secret scan (gitleaks)`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `timeout-minutes` | number | `10` | Job time limit |

### `shared-cd-docker-publish.yml`

Origin: the `docker-publish` job of `cd-staging.yml`/`cd-production.yml`
(terracore-front, okroot-back) and `docker-build-push.yml` (okroot-front).
Builds the image, scans it with Trivy (optional; fails on any fixable
CRITICAL), pushes it to GHCR with the tag `<tag-prefix>-<full sha>` and, if
asked, sends a `repository_dispatch`. With the scan on, the pushed image is
the one that was scanned. The dispatch only fires on a real `push`, never on
`workflow_dispatch`.

- Caller trigger: `push` to `stg` or `main` (and `workflow_dispatch` to republish without notifying)
- Permissions (granted by the caller): `contents: read` and `packages: write`; `contents: write` when dispatching to the same repo without `DISPATCH_TOKEN`
- Secrets: `BUILD_ARGS` (optional, `KEY=value` per line) and `DISPATCH_TOKEN` (optional; required when `dispatch-target-repo` is another repo). Without `DISPATCH_TOKEN` it uses the workflow token
- Dispatch payload: `{"environment": <tag-prefix>, "image_digest": ..., "image": ..., "sha": ...}`, which covers the fields terracore-back, okroot-back and okroot-front read today
- Outputs: `image` (name:tag) and `digest`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `dockerfile` | string | `''` | Dockerfile path from the root. Empty uses `<context>/Dockerfile` |
| `context` | string | `'.'` | Build context |
| `image-name` | string | `''` | Full GHCR image name. Empty uses `ghcr.io/<owner>/<repo>` |
| `tag-prefix` | string | required | Tag prefix (`stg`, `main`, `prod`); also sent as `environment` in the payload |
| `build-args` | string | `''` | Non-secret build args, `KEY=value` per line |
| `environment` | string | `''` | GitHub Environment of the job. Empty binds none |
| `trivy-scan` | boolean | `true` | Scans the image before pushing it |
| `dispatch-event` | string | `''` | `repository_dispatch` event type. Empty sends nothing |
| `dispatch-target-repo` | string | `''` | `owner/repo` that receives the dispatch. Empty uses the current repo |
| `timeout-minutes` | number | `15` | Job time limit |

Environment secrets: the caller job cannot have `environment`, so it cannot
pass them. If the `environment` input is used and that Environment has
secrets named `BUILD_ARGS` or `DISPATCH_TOKEN`, those replace the ones the
caller passes.

```yaml
jobs:
  publish:
    uses: lumina-w/agents/.github/workflows/shared-cd-docker-publish.yml@v4
    permissions:
      # write: dispatch to the same repo with the workflow token
      contents: write
      packages: write
    with:
      tag-prefix: stg
      dockerfile: Dockerfile.prod
      dispatch-event: backend-image-updated
    secrets:
      BUILD_ARGS: |
        API_URL=${{ secrets.API_URL }}
```

### `shared-ci-astro.yml`

Origin: the `quality`, `build` and `e2e` jobs of terracore-page's `ci.yml`.
Three jobs for an Astro/pnpm repo: `quality` (Prettier, ESLint, `astro check`,
Vitest with coverage), `build` (needs `quality`, warns instead of failing when
`dist/` exceeds `dist-size-warning-mb`) and `e2e` (needs `quality`, installs
only the Chromium browser, no `--with-deps`, since the runner image already
ships Chrome's system libraries and a job cannot `apt-get`). `ci.yml`'s two
other jobs are not part of this workflow: secret scanning is unified on
`shared-gitleaks.yml`, and the dependency audit already has its own
`shared-security-audit-node.yml`.

- Caller trigger: `pull_request` to `main`, `stg`, `dev`
- Permissions: `contents: read` per job
- Checks: `Format & Lint`, `Build`, `E2E (Playwright)` and `gate` (green only when all of the above are, so branch protection can require one name regardless of stack)

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `node-version-file` | string | `'.nvmrc'` | Version file passed to `actions/setup-node` |
| `dist-size-warning-mb` | number | `50` | `build` logs a `::warning::` (does not fail) when `dist/` exceeds this size, in MB |
| `timeout-minutes-quality` | number | `15` | Time limit of the `quality` job |
| `timeout-minutes-build` | number | `15` | Time limit of the `build` job |
| `timeout-minutes-e2e` | number | `20` | Time limit of the `e2e` job |

### `shared-lighthouse.yml`

Origin: terracore-page's `lighthouse.yml`. Builds the site and audits it with
Lighthouse CI (`treosh/lighthouse-ci-action`, axe-core under the hood for the
accessibility checks), uploading the report to Lighthouse CI's temporary
public storage.

- Caller trigger: `push` to `main`/`dev` and `pull_request` to `main`/`stg`/`dev`
- Permissions: `contents: read`
- Check: `Lighthouse CI`

| Input | Type | Default | Use |
|---|---|---|---|
| `runner-label` | string | `''` | Runner label |
| `node-version-file` | string | `'.nvmrc'` | Version file passed to `actions/setup-node` |
| `config-path` | string | `'./lighthouserc.json'` | Lighthouse CI config, relative to the repo root |
| `timeout-minutes` | number | `15` | Job time limit |

### `shared-release.yml`

Implements the release standard of dev-standards
([`docs/release-standard.md`](https://github.com/lumina-w/dev-standards/blob/main/docs/release-standard.md)).
Checks out the release branch, runs every check and only then creates the tag
`v<version>` on the branch head and the GitHub release in one call, with the
changelog section of that version as the notes. If a check fails, nothing is
created. The checks: the version is valid semantic versioning without the `v`,
the `prerelease` input matches the version suffix, the tag and a release for it
do not exist yet, the version file (when set) carries the same version, and the
changelog has a non-empty `## [<version>]` section.

- Caller trigger: `workflow_dispatch` with a `version` input (and optional
  `prerelease` and `dry-run`), on the repo's default branch. The workflow always
  checks out `release-branch`, so the caller does not need to reach `main` first.
- Permissions: `contents: write` on the job only; it uses the workflow token.
- Check: `Tag and release`
- Outputs: `tag` and `url` of the created release (empty on a dry run).
- It does not start other workflows: a release created with the workflow token
  triggers none.

| Input | Type | Default | Use |
|---|---|---|---|
| `version` | string | required | Version without the `v`, e.g. `1.4.2` or `1.5.0-rc.1` |
| `prerelease` | boolean | `false` | Must be true when the version has a suffix, and only then |
| `dry-run` | boolean | `false` | Runs every check and reports what would be created, without creating anything |
| `changelog-path` | string | `'CHANGELOG.md'` | Changelog, relative to the repo root |
| `version-file` | string | `'package.json'` | JSON file whose `version` must equal the version. Empty skips the check |
| `release-branch` | string | `'main'` | Branch the tag is created on and the checks run on |
| `runner-label` | string | `''` | Runner label |

Minimal caller (`.github/workflows/release.yml`):

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version without the v, e.g. 1.4.2'
        required: true
        type: string
      prerelease:
        type: boolean
        default: false
      dry-run:
        type: boolean
        default: false
permissions: {}
jobs:
  release:
    uses: lumina-w/agents/.github/workflows/shared-release.yml@v4
    permissions:
      contents: write
    with:
      version: ${{ inputs.version }}
      prerelease: ${{ inputs.prerelease }}
      dry-run: ${{ inputs.dry-run }}
      changelog-path: docs/CHANGELOG.md
      runner-label: '["self-hosted", "build", "lumina-w"]'
```
<!-- shared-workflows:end -->

## Actions

### `start-postgres`

Composite action for test jobs that used a `services: postgres` container,
which needs a Docker daemon the organization's ephemeral runners do not have.
It runs `initdb` and `pg_ctl` from `/usr/lib/postgresql/<version>/bin` (the
lumina-w runner image and `ubuntu-latest` both ship PostgreSQL 16), with the
cluster in `$RUNNER_TEMP`, listening on `localhost:<port>` with password
authentication, and creates the database.

```yaml
steps:
  - uses: lumina-w/agents/.github/actions/start-postgres@v4
    with:
      database: test_db
      user: test_user
      password: test_pass
```

| Input | Required | Default | Use |
|---|---|---|---|
| `database` | yes | | Database to create |
| `user` | yes | | Superuser to create |
| `password` | yes | | Its password. Test-only, never a real secret |
| `port` | | `'5432'` | TCP port on `localhost` |
| `version` | | `'16'` | Major version whose binaries to use |

## Relationship with dev-standards

`@lumina-w/dev-standards` is the source of truth for the commit rules and the
branch chain; this repo runs checks in CI.

- **Two ways to resolve `extends: ["@lumina-w/dev-standards/commitlint"]`.**
  `install-dependencies: true` runs `npm ci` in the caller repo, loading
  `DEV_STANDARDS_DEPLOY_KEY` with `webfactory/ssh-agent`, so the config
  resolves from the repo's own `node_modules`. The caller has to carry a
  `package.json`, a lockfile and the deploy key, and every job in that repo
  that runs `npm ci` now needs the key too, whether it touches commit rules
  or not; only terracore-back does this, since it already has a `package.json`
  for no other reason. `dev-standards-ref: vX.Y.Z` instead clones
  `dev-standards` at that tag straight into the scratch install (the same
  isolated temp dir the default path already uses for `@commitlint/cli`),
  with the same deploy key, so the caller needs only the one-line
  `.commitlintrc.json` and never touches its own `package.json` or its other
  jobs' `npm ci`/`pnpm install`. Prefer this for a new consumer.
  The remaining repos without either input lint against a local copy of the
  rules as they were in dev-standards 0.1.0: no `billing` scope, and the
  ` (#<number>)` a squash merge appends counts toward the 72 characters
  (0.2.0 stops counting it).
- `shared-validate-pr-base.yml` is the server-side counterpart of the
  dev-standards push chain: it checks the PR direction `dev -> stg -> main`.
- `shared-release.yml` implements `docs/release-standard.md`: the version format,
  the changelog shape and the checks come from there; change the standard first,
  then the workflow.

## Versioning

Callers always point to the major tag (`@v4`), never to `main`. Each merged
change that alters what callers run gets an immutable `v4.Y.Z` tag and moves
`v4` to it, the same scheme as GitHub's official actions. Docs-only changes
ride with the next release.

| Kind of change | Examples | What to do | Callers |
|---|---|---|---|
| Compatible | Fix, removing a step, a new optional input or secret | Tag `v4.Y.Z` and move `v4` | No change |
| Breaking | New required input or secret, renaming or removing an input | New major tag (`v5`, `v5.0.0`) | A PR in each caller to move to `@v5` |

Rollback: move `v4` back to the previous `v4.Y.Z`.

Tags today: `v4.0.0`, `v4.1.0`, `v4.1.1`, `v4.2.0`, `v4.2.1`, `v4.3.0`,
`v4.4.0` (shared workflows, agents#13), `v4.5.0` (agents#14), `v4.6.0`
(commitlint without a container, Dependabot PR titles and docs-sync fixes;
agents#16 to #20), `v4.7.0` (Dependabot commits in commit lint, agents#21,
current `v4`).
The older majors `v2` and `v3` still exist; no caller in the organization uses
them.

The full procedure is in [`CONTRIBUTING.md`](CONTRIBUTING.md).
