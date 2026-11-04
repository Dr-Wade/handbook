# Stack

The default for a new project and the direction for existing ones. Deviate when the project has
a reason, and write the reason in its README or agent notes. Versions are majors: upgrade minors
freely, majors after reading the changelog, one major at a time.

A new project starts from the [starter](https://github.com/Dr-Wade/starter) template, never from
an empty folder: it is this page as running code, with a small example through every layer, and
its agent notes walk through turning it into a project. When this page changes, the starter
changes with it.

## Defining traits

1. **TypeScript end to end, strict, ESM.** One language from the database schema to the browser.
2. **Types cross the wire.** Zod schemas in a shared package are imported by the API and the
   client; the API's OpenAPI spec generates the client SDK (`@hey-api/openapi-ts`). No
   hand-written API client. The API validates with NestJS's built-in Standard Schema support
   (`@Body({ schema })`, `StandardSchemaValidationPipe`) and documents with
   `@ApiOkResponse({ standardSchema })`: no DTO classes, no class-validator, no nestjs-zod.
3. **One auth system, mounted once.** `better-auth` inside the API, `better-auth/vue` in the
   client, cookie sessions. Nothing in `localStorage`.
4. **Slow work leaves the request.** OCR, mail sync, webhooks and batch jobs go through BullMQ to
   a separate worker process. The API enqueues, the worker processes.
5. **Plain pnpm workspaces.** No Nx, no Turbo: `pnpm --filter` and `pnpm -r` are enough at this
   size, and one less tool to keep in CI.
6. **Same-origin, no runtime URLs.** The client calls `/api/...`; the dev server proxies it and
   the reverse proxy routes it in production. No API URL baked into a build.

## Versions

| Area | Choice | Major |
|---|---|---|
| Runtime | Node | 26 (`engines`, `.nvmrc`) |
| Package manager | pnpm, pinned in `packageManager` | 12 |
| Language | TypeScript, `strict`, ESM (6.0 at the root, see below) | 7 |
| API | NestJS on Express | 12 |
| Database | PostgreSQL + Prisma (`prisma-client` generator, pg adapter) | 18 / 7 |
| Queues | BullMQ + Redis, through an ioredis client | 6 / 8 |
| Auth | better-auth | 1.7 |
| Validation | Zod | 4 |
| Client | Vue (Composition API, `<script setup>`) + Vite | 3.5 / 8 |
| Routing, state | vue-router, Pinia (UI state only) | 5 / 4 |
| Server state | TanStack Query; TanStack DB when the app works offline | 5 / 0.x |
| Styling | Tailwind via its Vite plugin (no PostCSS) | 4 |
| Components | PrimeVue (Aura preset, `definePreset`), primeicons | 4 / 7, held |
| i18n | vue-i18n | 11 |
| Unit tests | Vitest | 5 |
| E2E | Playwright | 1.x |
| Lint | ESLint flat config, [`@drwade/eslint-config`](https://github.com/Dr-Wade/eslint-config) | 10 |
| Errors | Sentry (`@sentry/nestjs`, `@sentry/vue`); no DSN, no calls | 11 |
| Logs | pino to stdout, `nestjs-pino` in Nest | 10 |
| Files | `@aws-sdk/client-s3` against any S3-compatible store | 3 |
| Mail | nodemailer, behind an `isConfigured()` check | — |
| Hosting | Docker images on Dokploy, see [`deploy.md`](deploy.md) | — |

### Held back on purpose

- **PrimeVue 4, `@primeuix/themes` 2, primeicons 7.** Their next majors (PrimeUI) are no longer
  MIT: a licence key per project, free only for small organisations. Stay on the last MIT majors
  until a project decides to license or to change library; Renovate rules keep them there.
- **TypeScript 7 needs three workarounds**, each to drop when its tool catches up:
  the root `typescript` stays on 6.0 because typescript-eslint loads it and supports < 6.1;
  `vue-tsc` runs through a small script that hands it `@typescript/typescript6`, since
  TypeScript 7 ships no JavaScript compiler to patch; `@hey-api/openapi-ts` is a `next`
  snapshot until a stable release supports TypeScript 7.

## Layout

```
repo-root/
├── package.json            workspace scripts only
├── pnpm-workspace.yaml     packages/* services/* (or apps/*)
├── docker-compose.yaml     local Postgres, Redis, S3 store, on non-standard ports
├── packages/
│   ├── schemas/            Zod schemas shared by API and client
│   ├── database/           Prisma schema, client, migrations
│   ├── queues/             queue names and job types (only with a worker)
│   └── logger/             pino
├── services/
│   ├── api/                NestJS HTTP server, global prefix /api, Swagger at /api/docs
│   ├── worker/             NestJS application context, no HTTP (only with slow work)
│   └── client/             Vue SPA
├── docs/                   VitePress, when the project has users who read docs
└── .github/workflows/      see ci.md
```

A small app drops `worker`, `queues` and `docs`. A single-tenant app drops the tenant layer.

## Key choices

### Backend
- **One NestJS module per domain resource.** Controllers stay thin; anything an agent, a CLI or
  a job should also reach is a service method.
- **Config is a Zod-validated service read lazily**, on first use, not at module registration:
  tests can then build the module with fakes and no environment.
- **Multi-tenant only when the domain is.** `User` (better-auth) → `Member` (per tenant) →
  `MemberProfile`; every business row carries the tenant id, enforced in one guard plus a CASL
  ability factory, never per controller.
- **Optional integrations no-op when unconfigured** (mail, Sentry, analytics): the app boots and
  its tests run with an empty environment.
- **`@Inject(Token)` on every constructor parameter.** Vitest's transform emits no decorator
  metadata. The API itself runs its `tsc` output, in development too: Swagger reads route
  parameters from `design:paramtypes`, which only `tsc` emits.
- **One Redis client per process, and a queue prefix of the project's own.** BullMQ 6 in an ES
  module needs a client instance, which the service creates and closes on shutdown. Every queue
  and worker passes the project's prefix: two projects on one Redis with the default `bull`
  prefix take each other's jobs.
- **Sentry collects no request data.** Sentry 11 sends cookies, headers and bodies unless
  `dataCollection` turns them off; errors and stack traces are what is needed.

### Frontend
- **Server state is not Pinia state.** Fetching, caching and invalidation go through TanStack;
  Pinia holds UI state only.
- **Forms** use a small `useForm()` composable with Zod validation, the same schemas the API uses.
- **Runtime config comes from `/api/config`**, not from build-time env: one image runs in every
  environment.
- **A PWA only when the app is used on phones or offline**, and then it gets a manifest, icons, a
  service worker that never caches API responses by accident, and an offline page.

### Local development
- `pnpm install`, `cp .env.example .env`, `pnpm db:up`, `pnpm db:migrate`, then one `dev:*`
  script per service. One `.env` at the root shared by every service.
- Nothing listens on a fixed, common port: several worktrees and agents run at once.

## Known deviations

| When | Use instead | Why |
|---|---|---|
| The product is a realtime app on Firebase already | Firestore + Cloud Functions, emulator in tests | Rewriting the backend is not worth it; keep the client side of this stack |
| A tool runs on one machine for one operator | `node:sqlite` instead of Postgres + Prisma | No server to run or back up |

## Things to avoid

- **Observability "just in case".** No OpenTelemetry, Datadog or log shippers until there is
  traffic that needs them: stdout logs plus Sentry first.
- **Type-checking inside `vite build`.** Run `vue-tsc --noEmit` / `tsc --noEmit` as its own step,
  once.
- **Proxying `/api` from the client's nginx.** The reverse proxy routes it; the client container
  serves static files only.
- **Two versions of `@tanstack/query-core` in the tree.** Pin it in `pnpm.overrides` when using
  TanStack DB: the collection integration breaks on duplicates.
- **Coverage thresholds on wiring** (`main.ts`, config, auth glue, health). Test services, not
  the bootstrap.
- **String-built shell commands.** Spawn subprocesses with `execa` and an argument array.
- **Secrets in config files.** Config holds references; values come from the environment or the
  secret store.
