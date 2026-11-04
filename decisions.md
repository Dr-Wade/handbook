# Decisions

One dated entry per decision: what was decided and why. Newest last. Entries are not edited
after the fact; a decision that is reversed gets a new entry.

- **2026-11-03 — A handbook, public and personal.** Conventions were spread over each project's
  docs and had drifted apart (three different CI shapes for the same stack). They live here now,
  in one repo outside any org: the projects span several GitHub orgs, and a public repo can be
  read from all of them and could host reusable workflows that repos in any org can call
  (a private repo's workflows can't be called from another org). Anything naming a server, host,
  customer or person stays in the private infra repo.

- **2026-11-03 — The PR check takes under 2 minutes; the full suite runs on push and gates the
  deploy.** PR checks had grown to run every suite with coverage, E2E, Docker smoke builds and
  browser tests on every push, 5–15 minutes each, and PR runs were the bulk of the Actions bill.
  The PR check is now one job running only what the PR can break, without coverage; the full
  suite runs on push to `develop` / `main` and the deploy `needs:` it. Large projects work the
  same way (Rust, Kubernetes, Chromium, Node.js, CPython, GitLab): fast checks per PR, the full
  suite after merge, a red main stops the line.

- **2026-11-03 — No mandatory full gate before merge; an opt-in `full-ci` label instead.** A
  `ready-to-merge` gate running the full suite on every PR before merge was considered and
  dropped as too heavy for the risk: GitHub's merge queue needs Enterprise Cloud on private
  repositories and required checks need a paid plan, so the gate could not be enforced anyway,
  and the deploy gate already stops a broken commit from shipping. Risky PRs get the full suite on
  demand with the `full-ci` label.

- **2026-11-03 — Coverage is measured on push, not on PRs.** Instrumentation costs 10–20% of a
  test run, and a threshold only means something on the full suite, which the PR check no longer
  runs. Thresholds fail the push and block the deploy.

- **2026-11-03 — GitHub-hosted runners; build once, deploy by image.** Self-hosted runners were
  weighed against the minute budget: after the PR-check change every org fits its free minutes,
  and a self-hosted runner adds a machine to patch and a place for untrusted code to run. They
  are reconsidered only past the thresholds in `deploy.md`. Deploys pull a tested image from
  GHCR; Dokploy never builds from source, and production gets the image staging tested.

- **2026-11-03 — E2E: only what needs a browser, under 5 minutes, against the built app.** One
  project's Playwright suite took 26 minutes for 307 tests: one worker (Playwright's default on a
  2-vCPU private runner), the Vite dev server reloading every module in each test's fresh
  context, about 5 seconds per test with no sleeps anywhere. Locally the same tests ran 2.5×
  faster per test against `vite build` + `vite preview`, and 6× with 4 workers. Most of them
  checked one page against a mocked API, which is a component test's job. Hence the rules in
  `ci.md` § E2E: a budget, the built app, explicit workers, no retries, selection by app when the
  API is mocked, and Playwright only for what needs a real browser.

- **2026-11-03 — Merges in a row share a full-suite run; stacks merge from the top.** An agent
  that works through several issues overnight leaves several PRs, and merging them one after
  another ran the full suite once per merge (13–15 billed minutes each on one project), most of
  them for commits nobody would deploy. A concurrency group without `cancel-in-progress` keeps
  one run going and one waiting, the newest; a cancelled run counts as no verdict so a red-branch
  watcher can't be cleared by it. GitHub's stacked pull requests (generally available since
  2026-10-06) land a whole stack in one push when its top PR is merged.

- **2026-11-04 — New projects start from a template repository, not a CLI or shared packages.**
  Projects that started from an empty folder picked their own stack and had to be moved onto
  this one later. Three remedies were weighed. Versioned shared packages keep apps in sync, but
  they couple projects in different organisations and need release discipline: the one shared
  package that existed had gone stale and was overridden in every consumer. A scaffolding CLI is
  one more tool to maintain. A template repository (`Dr-Wade/starter`) is copied once and owned
  by the project from then on; projects may drift, which is acceptable, and a fix found in one
  project goes back into the starter so the next project starts with it. Its example runs through
  every layer because an agent follows working code more faithfully than prose; a fresh agent
  given only the starter scaffolded a project with it, and the gaps it hit were fixed in the
  starter's notes.

- **2026-11-04 — The stack moves to the current majors: Node 26, pnpm 12, TypeScript 7, NestJS 12,
  BullMQ 6, Vitest 5, ESLint 10, Sentry 11.** Taken together in the starter rather than project by
  project. TypeScript 7 costs three workarounds (root TypeScript 6 for typescript-eslint, a
  `vue-tsc` shim, a hey-api snapshot), recorded in `stack.md` with the condition that removes
  each. Prisma stays on 7: npm's `latest` tag points at an 8.0 release candidate.

- **2026-11-04 — Validation through NestJS's built-in Standard Schema support.** NestJS 12 validates
  parameters against a Zod schema given to the decorator and `@nestjs/swagger` 12 documents it,
  so the shared Zod schemas serve the API, the OpenAPI document and the client's forms with no
  DTO classes. class-validator would have meant a second definition of every contract, and
  nestjs-zod no longer adds anything (and had not declared NestJS 12 support).

- **2026-11-04 — PrimeVue held at 4.** PrimeVue 5 and its themes and icons moved to a commercial
  licence with a key per project, free only below a revenue, headcount and funding threshold
  that several projects exceed. Version 4 is MIT and what every project runs. Moving off it is a
  choice between licensing and another component library, to be made when 4 stops being enough.
