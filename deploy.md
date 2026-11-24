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
- **Dokploy pulls the image**; it never builds from source on the server. A build there competes
  with the running apps for CPU and disk, and nothing tested what it built. Each service is one
  Dokploy application: source *Docker*, image `ghcr.io/<owner>/<repo>/<service>:<environment
  tag>`, registry credentials of a service identity (never a person's login), and *Autodeploy*
  off: the pipeline triggers the redeploy, Dokploy does not watch the branch.
- **The shared workflows do the generic steps** ([`ci.md`](ci.md#workflow-files)):
  `publish-image.yml` builds and pushes one image, `promote-image.yml` re-tags a tested one, and
  `deploy-dokploy.yml` redeploys and waits on the 1-vCPU runner, since it only polls.
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

## Redeploying without downtime

Dokploy runs each application, and each database it manages, as a Swarm service. When an app's
*Update Config* is left empty, Dokploy deploys it **start-first** with rollback on failure: the
new container starts, and the old one is stopped as soon as the new one is *running*. Running is
not ready. Without a health check, the old container is stopped while the new one is still
booting, and requests fail for as long as the boot takes, migrations included: 10 seconds of
502s on a small API, webhooks and all.

- **The image carries a `HEALTHCHECK`** that asks the app itself whether it is ready
  (`/api/health` for an API, the index page for a client container). Swarm then waits for
  *healthy* before stopping the old container and sends it no traffic until then. It is in the
  image, not in Dokploy, so it is versioned with the code. Probe with something the image has:
  `node -e "fetch(…)"` in a Node image, `wget` in an Alpine-based nginx one, never `curl` by
  default. Give it a start period that covers the migrations.
- **HTTP services keep start-first.** Dokploy's default already is; setting it explicitly
  (*Advanced → Swarm Settings → Update Config*,
  `{"Parallelism": 1, "Order": "start-first", "FailureAction": "rollback", "Monitor": 30000000000}`)
  documents the choice. A new container that never gets healthy is rolled back instead of taking
  over.

**Anything that owns data is set to stop-first, explicitly**:
`{"Parallelism": 1, "Order": "stop-first", "FailureAction": "pause"}`, and the same `Order` in
*Rollback Config*. Dokploy's start-first default applies to databases, Redis, object storage and
apps with a data volume too, so on every redeploy two processes open the same files for a few
seconds. A Postgres data directory, a SQLite file, an AOF or a search index does not survive that
reliably. The same goes for singletons that must never run twice: a sync service that keeps a
cursor, a scheduler, a bot polling for updates.

**A service that keeps state keeps it on a named volume.** An image that declares `VOLUME`
(Redis, Postgres) gets an anonymous volume per container otherwise, and the next container
starts empty. Redis for BullMQ also runs with `--appendonly yes`.

**The process stops when asked.** Swarm stops a container with `SIGTERM` and, 10 seconds later,
`SIGKILL`. A process running as PID 1 has no default action for `SIGTERM`, so unless it handles
the signal it is killed mid-request ("Exited (137)"). The entrypoint `exec`s the process
(`sh -c "migrate && exec node dist/main.js"`, not `… && node dist/main.js`, where the shell gets
the signal), and the app handles `SIGTERM` by closing (finishing in-flight requests and the
current job) and then exiting. A Nest app calls `app.close()` and then `process.exit()` itself:
Nest's `enableShutdownHooks()` re-raises the signal after closing, PID 1 ignores it, and any
timer or open handle left (a heartbeat, a metrics exporter) keeps the process up until the
`SIGKILL`. Dokploy's *Command* field is split on spaces into the entrypoint, without a shell:
`docker-entrypoint.sh redis-server --appendonly yes` keeps the image's entrypoint and PID 1;
a leading `exec` fails.

With start-first, the old and new versions of an app run side by side for a few seconds against
the same database, which is one more reason migrations stay backward-compatible (see below).

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
