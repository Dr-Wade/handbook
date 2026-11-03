# CI

Fast feedback on every PR, the full suite after merge, and nothing deploys unless the full suite
passed. This is how most large projects work (Rust, Kubernetes, Chromium, Node.js, CPython,
GitLab): a mandatory full run on every PR is the exception, not the norm.

## The model

| When | What runs | Budget |
|---|---|---|
| Every push to a PR | **The PR check**: lint, typecheck and tests of what the PR touches, no coverage | under 2 minutes, one job |
| A PR labelled `full-ci` | The full suite on that PR, on demand | as long as it takes |
| Every push to `develop` / `main` | **The full suite**: every test with coverage thresholds, E2E, image builds | as long as it takes |
| Deploy | Only after the full suite passed on that commit (`needs:`) | — |

**A red `develop` or `main` stops the line.** The deploy is already blocked by `needs:`; the next
work is reverting or fixing forward, before any new PR merges. Revert first when one commit is
clearly to blame and the fix is not obvious.

## The PR check

- **One job.** GitHub bills every job rounded up to a whole minute, so three 20-second jobs cost
  three minutes and one 60-second job costs one. Split jobs only when one of them needs a
  different runner or a service container the others don't.
- **Only what the PR can break.** A "what changed" step diffs the PR against its base and sets
  flags; each later step runs only for its area. Changes to install inputs (lockfile,
  `package.json`, workspace file, tsconfig, ESLint config, workflows) run everything, still
  without coverage.
- **Lint the changed files only.** ESLint rules without type information are per file.
- **Typecheck what the PR touches**, or all of it when that is cheap; it catches what the
  affected-tests selection misses.
- **Affected tests only.** In order of preference: the tool's import graph
  (`vitest run --changed HEAD^1`), the workspace graph (`pnpm --filter "...[<base>]"`), or a
  directory mapping in the workflow when neither works (virtual modules, `.vue` files the graph
  walker can't read).
- **No coverage.** Instrumentation costs 10–20% and a threshold is only meaningful on the full
  suite; coverage is measured on push.
- **No E2E, no image builds, no browser component tests.** They run on push.
- **Cancel superseded runs**: a new push makes the run in flight worthless.
- **Skip docs-only changes** with `paths-ignore`.
- **Databases only when needed.** A Postgres or emulator service container starts only when a
  suite that needs it runs; otherwise it is a second job, path-filtered.

### Recipes

The merge commit and its parents, so `HEAD^1` is the base:

```yaml
on:
  pull_request:
    paths-ignore: [ 'docs/**', '**/*.md' ]

concurrency:
  group: pr-check-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 2
```

What changed, as step outputs. File names never go into a script through `${{ }}`: a PR names
its own files.

```yaml
      - name: What changed
        id: changed
        run: |
          files=$(git diff --name-only HEAD^1 HEAD)
          touched () { printf '%s\n' "$files" | grep -qE "$1"; }
          all=false
          if touched '^(pnpm-lock\.yaml|pnpm-workspace\.yaml|(.*/)?package\.json|tsconfig[^/]*\.json|eslint\.config\.js|\.github/workflows/)'; then all=true; fi
          echo "all=$all" >> "$GITHUB_OUTPUT"
          if [ "$all" = true ] || touched '^services/api/'; then echo "api=true"; else echo "api=false"; fi >> "$GITHUB_OUTPUT"
          printf '%s\n' "$files" | grep -E '\.(ts|vue|js|mjs|cjs)$' \
            | while IFS= read -r f; do [ -f "$f" ] && printf '%s\n' "$f"; done > "$RUNNER_TEMP/lint.txt" || true

      - name: Lint changed files
        run: xargs -r -d '\n' pnpm exec eslint --max-warnings 0 --no-warn-ignored -- < "$RUNNER_TEMP/lint.txt"
```

`xargs -r` matters: without it an empty list runs ESLint on the whole repo.

Related tests with Vitest:

```bash
pnpm exec vitest run --changed HEAD^1 --passWithNoTests
```

Vitest reruns everything when a file in `forceRerunTriggers` changed (by default any
`package.json` and Vitest/Vite config). Files the import graph can't see (a rules file read
from disk, a package consumed through its build) need their own flag that runs the whole suite.

`vitest run --no-isolate` roughly halves the time of a suite whose tests don't leak global
state. Try it per package; keep it only where the suite passes the same both ways.

## The `full-ci` label

For a PR that is risky enough to deserve the full suite before merge: a migration, a large
refactor, a dependency major. Adding the label runs it; pushes while the label is on run it
again. It costs nothing unless used.

```yaml
on:
  pull_request:
    types: [ labeled, synchronize ]

jobs:
  full:
    if: >-
      (github.event.action == 'labeled' && github.event.label.name == 'full-ci') ||
      (github.event.action == 'synchronize' && contains(github.event.pull_request.labels.*.name, 'full-ci'))
    uses: ./.github/workflows/build.yml
```

A project may also escalate by path (database migrations run the full suite without a label),
as long as ordinary PRs stay under 2 minutes.

## The full suite

- Every test, with **coverage thresholds**. A threshold fails the push, not the PR.
- E2E (Playwright) against the built app.
- The images the deploy will ship, built once (see [`deploy.md`](deploy.md)).
- It is a reusable workflow (`build.yml` with `on: workflow_call`) that the deploy workflow
  calls, so the gate and the deploy can't drift apart.

## Coverage

Coverage is a signal, not a merge gate. Thresholds live in the full suite; a PR can show a
coverage report as information. Exclude wiring (bootstrap, config, auth glue, health) from the
measured set rather than writing tests for it.

## Flaky tests

A flaky test is fixed, not retried and not marked slow. A test that needs a timeout or a sleep to
pass is wrong: inject the clock. A quarantined test carries the issue number and the reason on
the line above it.

## Minutes

Private repositories on a free org get 2,000 Actions minutes a month; public repositories are
free. The number of runs is what drives the bill, and runs scale with pushes:

- **Commit often, push once per finished unit of work.**
- **Every job costs at least a minute**, even one that does nothing: fewer, longer jobs beat many
  short ones.
- **A workflow that only needs to run for some paths says so** with `paths` / `paths-ignore`.
- **Scheduled workflows are budgeted** like pushes: a nightly run is 30 runs a month.
- **An agent asks before a push that starts a billed run**, and says what it costs.

## What we don't use

- **GitHub merge queue**: on private repositories it needs GitHub Enterprise Cloud.
- **Required status checks / rulesets**: on a free org's private repositories they need a paid
  plan. The deploy gate (`needs:`) is the hard gate instead; merges are guarded by review and,
  where an agent merges, by the agent's merge policy.
- **Self-hosted runners for PR checks.** See [`deploy.md`](deploy.md#runners).
