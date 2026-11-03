# Deploy

GitHub Actions builds and tests; Dokploy runs the containers. A deploy ships an image that passed
the full suite, and nothing else.

## Environments

| Branch | Environment | Image tags |
|---|---|---|
| `develop` | staging | `dev`, `dev-<sha>` |
| `main` / `master` | production | `latest`, `<sha>` |

A project with a single environment deploys from `main` only. Releasing to production is a PR
from `develop` into `main` / `master`.

## The pipeline

```
push ─▶ build.yml (full suite) ─▶ detect changed services ─▶ publish-service.yml per service
                                                              build image ─▶ push to GHCR ─▶ Dokploy redeploy, wait
```

- **The deploy `needs:` the full suite.** No path, manual trigger included, deploys a commit whose
  full suite did not pass. A `workflow_dispatch` input that skips tests or E2E is not allowed.
- **Only the services that changed are published** (`dorny/paths-filter` or the same "what
  changed" step as the PR check). A change to a shared package publishes every service that
  imports it.
- **Images go to GHCR** as `ghcr.io/<owner>/<repo>/<service>`, tagged with the environment and
  the commit, built with Buildx and the GHA layer cache (`cache-from: type=gha,scope=<service>`).
- **Dokploy pulls the image**; it never builds from source on the server. Each service is one
  Dokploy application whose source is the image's environment tag.
- **The redeploy is triggered with
  [`Dr-Wade/dokploy-deploy-action`](https://github.com/Dr-Wade/dokploy-deploy-action) and
  `wait_for_deployment: "true"`**: the pipeline goes red when the deployment itself fails, not
  only when the trigger does.
- **A publish is never cancelled half way** (no `cancel-in-progress` on deploy workflows): it
  would leave some services on the new version and some on the old. A newer push replaces the
  run still waiting instead (see [`ci.md`](ci.md#the-full-suite)).

## Build once

The image that passed staging is the image that reaches production. Promoting means re-tagging
the tested image (`docker buildx imagetools create`, or Dokploy pointed at the commit tag), not
rebuilding it from `main`: a rebuild can pull a different base image or dependency and ships
something nothing tested. Projects that still rebuild on `main` move to promotion when their
deploy workflow is next touched.

## Configuration

- **One image for every environment.** Runtime configuration comes from the environment and,
  for the client, from `/api/config`. The only build argument is the app version.
- **Secrets** live in GitHub Actions secrets (`DOKPLOY_TOKEN`) and in Dokploy's environment for
  the app; **identifiers** in Actions variables (`DOKPLOY_URL`, `DOKPLOY_<SERVICE>_APP_ID`).
- **Routing** is the reverse proxy's job: `<domain>/api/*` to the API, everything else to the
  client container.

## Rolling back

Redeploy the previous image: point the application at the previous `<sha>` tag, or use
Dokploy's rollback. Then revert on the branch so the next push doesn't ship the bad commit again.
A database migration is written so the previous version still runs against the migrated schema
(add, then backfill, then remove in a later release).

## Runners

GitHub-hosted runners (`ubuntu-latest`) for everything. They are clean every run, need no
upkeep, and the PR check is designed to fit the free minutes.

A self-hosted runner is worth it only when, for two months running, an org goes past its free
minutes or deploy runs take longer than about 10 minutes. Even then:

- **Dedicated machine**, amd64, used for nothing else. Never a production Dokploy host (a build
  competes with the apps for CPU and disk, and a compromised job lands on production), never a
  developer machine that also runs agents.
- **Ephemeral runners** (a fresh runner per job), registered to private repositories only: a
  public repo's PR could run code on it.
- **Deploy-time jobs first.** The PR check stays on GitHub-hosted runners.

## Other targets

- **Firebase** projects deploy Hosting and Functions from the same gated pipeline, after the
  full suite with the emulator.
- **Desktop and mobile builds** run on release tags or on demand, not on every push: they are
  the most expensive runners (macOS bills at 10× the Linux rate).
