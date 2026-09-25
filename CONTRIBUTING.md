# Contributing to agents

Every workflow in this repo runs inside the organization's product and docs
repos through the `v4` tag. Moving that tag changes CI for all of them at
once, so changes follow the steps below.

## Workflow

1. Branch from `main`: `<type>/<short-kebab-case-description>`.
2. Make the change. Keep the conventions of the existing workflows:
   - `on: workflow_call` only; triggers belong to the callers.
   - `permissions: {}` at the workflow level, minimal permissions per job.
   - Untrusted values (PR titles, labels, comments, branch names) go through
     `env:`, never straight into a `run:` script.
   - Third-party actions pinned to a major tag or a commit SHA.
   - `shared-*.yml` accept `runner-label` with the same `runs-on` expression
     (empty, a label, or a JSON array of labels).
   - Comments, step names and docs in English. No em dashes.
3. Update `README.md` in the same PR when inputs, secrets, triggers, checks
   or behavior change, and `AGENTS.md` when the dev agent changes. Keep the
   "Who calls what" table true.
4. Commit and title the PR as `<type>(<scope>): <description>` (scope `ci`
   for workflows, `docs` for documentation), 72 characters max.
5. Open a PR against `main`. Never push to `main`.
6. After the merge, release it (below). Until a tag moves, callers keep
   running the previous version.

## Testing a change before release

There is no CI in this repo, and a reusable workflow only runs when a caller
calls it. Before tagging:

1. Lint the YAML: `actionlint .github/workflows/<file>.yml` if available
   (it also checks expressions and shell), otherwise at least parse it
   (`python3 -c 'import sys, yaml; yaml.safe_load(open(sys.argv[1]))' <file>`).
2. Run it from a real caller without touching `v4`: on a throwaway branch of
   a consumer repo, point the caller at the branch or commit of this PR
   (`uses: lumina-w/agents/.github/workflows/<file>.yml@<branch-or-sha>`),
   trigger it, and check the result and the check names. Do not merge that
   caller change.
3. For a change to a check name, list the branch protection rules that
   require it in each consumer; they must be updated together with the tag.

Write what you ran and what you saw in the PR's test plan, and what you could
not run under "Not verified".

## Releasing

Callers pin the major tag. Each release gets an immutable `v4.Y.Z` tag and
moves `v4` to the same commit, like GitHub's official actions.

| Kind of change | Examples | Tag |
|---|---|---|
| Compatible fix | A bug fix, removing a step | `v4.Y.(Z+1)`, move `v4` |
| Compatible feature | A new optional input or secret, a new workflow | `v4.(Y+1).0`, move `v4` |
| Breaking | A new required input or secret, renaming or removing an input, a changed check name callers depend on | `v5.0.0` and a new `v5`; a PR in each caller to move to `@v5` |

```sh
git fetch origin --tags
git switch main && git pull
git tag v4.Y.Z            # on the merge commit
git push origin v4.Y.Z
git tag -f v4 v4.Y.Z      # move the major tag
git push -f origin v4
```

- Tags `v4.Y.Z` are never moved or deleted.
- Rollback: move `v4` back to the previous `v4.Y.Z` (`git tag -f v4 v4.Y.Z`
  and `git push -f origin v4`).
- A docs-only merge does not change what callers run. The precedent
  (agents#9) is that it has no tag of its own and rides with the next
  release.

Current tags: `v4.0.0` through `v4.5.0`; `v4` points to `v4.5.0`. `v2` and
`v3` still exist and have no callers in the organization.

## Secrets

Workflows read organization secrets (`CLAUDE_CODE_OAUTH_TOKEN`,
`DOCS_SOURCES_READ_TOKEN`, `PROMOTE_TOKEN`) or secrets the caller passes
(`DEV_STANDARDS_DEPLOY_KEY`, `GITLEAKS_LICENSE`, `BUILD_ARGS`,
`DISPATCH_TOKEN`). A new required secret is a breaking change. Declare every
secret under `on.workflow_call.secrets` with `required` set, and never print
one: check its presence as the existing workflows do (a step that writes
`present=true|false` to `$GITHUB_OUTPUT`).

## Who reads this repo

Besides the callers, `luminaw-docs` lists `agents` (branch `main`) as a
source of its daily docs sync, so the README and `AGENTS.md` on `main` feed
the organization's docs.
