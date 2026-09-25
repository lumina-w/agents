# AGENTS.md

Two things live in this file:

1. The rules for any coding agent (Claude Code or another) that edits this
   repo.
2. The reference of the **dev agent** (`.github/workflows/dev-agent.yml`),
   the Claude Code agent that the product repos run through this repo.

`README.md` has the catalog of every workflow, who calls it and how.
`CONTRIBUTING.md` has the change, test and release procedure.

## Working on this repo

- Every file here is a reusable workflow (`on: workflow_call` only) that
  production repos call through the `v4` tag. A change reaches them only when
  a tag moves, but a bad tag move breaks every caller at once.
- Branch from `main`, one PR per task, never push to `main` and never move a
  tag unless the task asks for a release.
- Commits and PR titles: `<type>(<scope>): <description>`, scope mandatory,
  72 characters max. Workflow changes use `ci`, docs use `docs`.
- Everything new is in English: comments, step names, docs, commits, PRs. The
  dev agent and docs-sync still print some runtime messages in Spanish; do not
  add new ones.
- No em dashes anywhere.
- Keep each job's `permissions` minimal and explicit, keep `permissions: {}`
  at the workflow level, pass untrusted values (PR titles, labels, comments)
  through `env:`, never interpolate them into `run:` scripts.
- Pin third-party actions to a major tag or a commit SHA, as the existing
  workflows do.
- A change to inputs, secrets, triggers or behavior updates `README.md` (and
  this file, for the dev agent) in the same PR.

## The dev agent

`dev-agent.yml` runs Claude Code (`anthropics/claude-code-action@v1`) inside
a consumer repo. Each consumer has a short `.github/workflows/claude.yml` that
calls it with `@v4` and passes what is specific to the repo: toolchain,
install command, done-criteria commands, extra tools, context docs and docs
repo. The logic exists only here.

Today eight repos call it: terracore-back, terracore-front, terracore-page,
okroot-back, okroot-front, okroot-page, blog-w and luminaw-page. Their inputs
are listed in the README.

### What it does and what it does not

| Does | Does not |
|---|---|
| Implements an issue and opens a PR against `dev` | Merge PRs |
| Iterates on its PR when someone comments `@claude` | Push to `dev`, `stg` or `main`, or force push |
| Fixes its PR when CI fails (2 rounds at most) | Promote `dev -> stg -> main` |
| Writes the repo's checkpoint, which the workflow commits to the same PR | Read `.env` files, install dependencies on its own, start servers |

Its work ends at the PR. Auto-merge (`automerge-dev.yml`), branch cleanup
(`delete-merged-branches.yml`) and promotion (`shared-promote-dev-to-stg.yml`,
then a person for `stg -> main`) belong to the repo's workflows.

### Triggers and modes

| Event | Mode | Size | Default model | Turns |
|---|---|---|---|---|
| Label `dev-ft-ready` added to an issue | implement | full | `opus` | 80 |
| Label `dev-ft-small` added to an issue | implement | small | `sonnet` | 40 |
| Comment containing `@claude` on a PR | iterate | small | `sonnet` | 40 |
| The repo's `CI` workflow fails on a PR (`workflow_run`) | fix | full | `opus` | 80 |

The `opus` and `sonnet` aliases resolve to the latest model that
`CLAUDE_CODE_OAUTH_TOKEN` allows. `model`, `small_model`, `max_turns` and
`small_max_turns` override them per repo.

Trigger rules (checked in the first step, before anything is checked out):

- Only someone with `write`, `maintain` or `admin` permission on the repo can
  trigger it, by labeling or commenting.
- Events sent by bots are ignored, to avoid loops.
- One run at a time per issue or PR: concurrency group
  `dev-agent-<repo>-<issue or PR branch>`, and a new event waits instead of
  cancelling the running one.
- iterate and fix only act on open PRs from the same repo (never forks).
- fix only acts on PRs the agent opened (label `dev-agent`). Each round adds
  the label `agent-fix-<n>`. When the rounds reach `max_fix_rounds` (2), it
  comments on the PR asking for human review and stops. A person continues
  by commenting `@claude` with instructions.
- A repo without a workflow named `CI` (today luminaw-page) has no fix mode.

### The nine layers

| # | Layer | What it does |
|---|---|---|
| 1 | Trigger | Checks permissions, ignores bots, resolves the mode and limits the fix rounds |
| 2 | Toolchain | Sets up Python, npm or pnpm with the repo's versions (with dependency cache) and runs `install_command` before the agent starts |
| 3 | Environment | Exports the repo's `env_vars` for local checks. Never secrets |
| 4 | Context | The agent reads `CLAUDE.md`, `AGENTS.md`, the `context_files`, the skills in `.claude/skills/` and, read-only, the project docs repo (`<project>-docs`) |
| 5 | Commands | It can only run the repo's done-criteria commands (`verify_commands`) plus a few extras (`extra_allowed_tools`). The narrowest first; the full list once before every push |
| 6 | Guardrails | Denies merge, force push, pushes to `dev`/`stg`/`main`, `reset --hard` and reading `.env` files |
| 7 | Feedback | The agent can read the PR's checks and failed logs; when CI fails after the run, fix mode picks it up |
| 8 | Checkpoint | The agent writes the full checkpoint to a temp file and the workflow commits it to the PR branch as `.claude/CHECKPOINT.md` |
| 9 | PR text and ready | Replaces em dashes in the PR title and body with ", ", then marks the draft PR ready |

GitHub branch protection is still the real barrier against pushes to
protected branches; the agent's guardrails are a second layer.

### Tools it may use

Always allowed: `Edit`, `Write`, and these commands: `git fetch`,
`git checkout -b`, `git switch -c`, `git switch`, `git add`, `git mv`,
`git commit`, `git push` (plain, `-u origin`, `origin`), `git status`,
`git diff`, `git log`, `git branch`, `gh issue view`, `gh pr create`,
`gh pr view`, `gh pr diff`, `gh pr checks`, `gh pr comment`, `gh run view`.

Added per repo: every line of `verify_commands` (as `Bash(<command>)`) and
every rule in `extra_allowed_tools`.

Always denied: `git push --force`, `-f`, `--force-with-lease`, any push to
`origin main`, `origin stg` or `origin dev` (with or without `-u`),
`git reset --hard`, `gh pr merge`, and reading `**/.env`, `**/.env.local`,
`**/.env.stg`, `**/.env.production`.

### Context and the project docs repo

- `CLAUDE.md`, `AGENTS.md` and each `context_files` entry are listed in the
  prompt only if they exist in the repo.
- With `docs_repo` and `DOCS_SOURCES_READ_TOKEN`, the workflow clones
  `lumina-w/<docs_repo>` into the runner's temp directory, outside the
  workspace, removes its `.git` and makes it read-only, so it can never be
  committed. Without the token the agent runs without product context.
- The repo's files win over any general habit: stack, versions, conventions,
  branch flow, commit and PR format, and done criteria come from them.

### What the prompt asks for

1. implement: branch off an up-to-date `dev` with the naming convention from
   `CLAUDE.md`. iterate/fix: stay on the PR branch.
2. Make the smallest change that solves the task, no unrelated refactors.
3. Write the full updated checkpoint to `$RUNNER_TEMP/agent-out/checkpoint.md`
   (starting from the current one, following `.claude/commands/checkpoint.md`
   if it exists). Never edit the checkpoint file in the repo.
4. Commit with the repo's convention and push only the working branch.
5. implement: open a draft PR against `dev` with the label `dev-agent`,
   referencing the issue, with the body sections `CLAUDE.md` requires; mark as
   done only what it ran and saw pass. iterate/fix: comment on the PR what it
   changed and why.
6. Stop at the PR: never merge, never push to `dev`/`stg`/`main`, never force
   push, never read `.env`.
7. If it cannot finish, explain why in a comment instead of guessing.
8. No em dashes or emojis in commits, PR titles or bodies.

### Checkpoint

- Claude cannot write inside `.claude/` (a protected path for its own tools),
  so the workflow commits the checkpoint after the agent finishes, to
  `checkpoint_file` (default `.claude/CHECKPOINT.md`, the organization
  standard).
- Commit message: `docs(docs): update checkpoint after #<n>`, authored as the
  last commit's author. It is pushed with `PROMOTE_TOKEN` so the PR's checks
  run again on it.
- If the repo still has `CHECKPOINT.md` at the root and not at the target
  path, it is moved with `git mv`.
- Without `PROMOTE_TOKEN`, or if the agent wrote no checkpoint, nothing is
  committed (the run logs a warning).
- It is the same file that the `SessionStart` hook, the `/checkpoint` command
  and `docs-sync.yml` read.

### Draft, ready and auto-merge

The agent opens the PR as a draft. The last step marks it ready
(`gh pr ready` with `PROMOTE_TOKEN`), after the checkpoint commit.
`automerge-dev.yml` ignores drafts and arms auto-merge on the
`ready_for_review` event, so the PR cannot be merged before the checkpoint
lands. If the agent fails, the PR stays in draft. Without `PROMOTE_TOKEN` it
also stays in draft.

### Labels

| Label | Set by | Meaning |
|---|---|---|
| `dev-ft-ready` | a person | Implement this issue (full model) |
| `dev-ft-small` | a person | Implement this issue (small model) |
| `dev-agent` | the agent, on its PRs | Marks PRs eligible for fix mode |
| `agent-fix-1`, `agent-fix-2`, ... | the workflow | Fix rounds already used on the PR (created on first use) |

### Job permissions and secrets

The caller job must grant `contents: write`, `pull-requests: write`,
`issues: write`, `id-token: write` and `actions: read`, and pass
`secrets: inherit`.

| Secret | Required | Use |
|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | yes | Claude authentication |
| `DOCS_SOURCES_READ_TOKEN` | no | Clones `<project>-docs` as read-only context |
| `PROMOTE_TOKEN` | no, but needed for the full flow | Commits the checkpoint and marks the PR ready, so checks and auto-merge run |

### Runner

`runner-label` (default `''`, `ubuntu-latest`) picks the runner, with the same
values as the `shared-*.yml` workflows (see the README). On the organization's
ephemeral runners the run holds the only job slot for up to `timeout_minutes`,
and the runner image must ship `gh` and the repo's toolchain.

### Where to look when it did not run

The first step prints why it skipped, in Spanish, and the run summary says
which mode ran on which issue or PR. Common reasons: the label is not one of
the two trigger labels, the person who triggered it lacks write access, the
event came from a bot, the PR is closed or from a fork, the PR lacks the
`dev-agent` label (fix mode), or the fix rounds are used up.
