# Bonafi monorepo scaffold

Greenfield agent brief. Start with an **empty folder**. Follow each section's official doc when anything is ambiguous; this file only pins our choices (names, ports, modes).

**Principle:** scaffold the foundation only. No `lib/<module>/` folders, no Drizzle schema, no Inngest functions, no example components. The module template lives in [bonafiai/docs](https://github.com/bonafiai/docs) (Structure, Types, Jobs) and is enforced through `.specify/memory/constitution.md`. Modules arrive only via approved specs.

**Apps:** `apps/web` (product), `apps/test` (internal).

---

## How to run

This file lives in the **docs** repo. The codebase it creates is a **sibling** repo, not inside `docs/`.

### Workspace layout

```text
~/www/
  docs/     bonafiai/docs (constitution, this brief)
  web/      bonafiai/web (monorepo you are scaffolding)
```

### Cursor setup

1. Open **`~/www`** in Cursor (multi-root: `docs` + `web`). Or open only `~/www/web` after the folder exists if you want the agent focused on one repo.
2. **Prerequisites on the machine:** [Bun](https://bun.sh), `git`, [uv](https://docs.astral.sh/uv/) (for spec-kit). WorkOS, PlanetScale, Inngest keys can stay empty in `.env.example` until you wire accounts.
3. Ensure **`~/www/web` does not exist** or is empty. This brief is greenfield. If an old `web` checkout is there, move it aside or use a fresh folder before starting.
4. In Agent mode, prompt along these lines:

   > Scaffold the Bonafi monorepo per `docs/.cursor/plans/scaffold.md`. Create everything under `web/` (sibling to `docs/`). Do not modify `docs/`. Work phase by phase; commit after each major phase; run the Acceptance checks at the end.

5. Optional: copy this file to `web/SCAFFOLD.md` first so the new repo carries its own brief. The agent can still read the docs copy for constitution links.

### What the agent should not do

- Scaffold inside `docs/` (wrong repo).
- Add `lib/<module>/` folders, Drizzle tables, Inngest functions, or example UI beyond the null `/home` route.
- Skip acceptance checks or leave `turbo run lint check-types test` red.

### After scaffold

- Push `web/` to `bonafiai/web` on GitHub (repo name matches folder until the legacy repo is archived).
- Open `web/` as its own Cursor window for day-to-day feature work.
- Run `/speckit.constitution` and transcribe Architecture pages from `docs/`.

---

## Phase 0: bootstrap from empty

Repo and folder name: **`web`** (matches `bonafiai/web` on GitHub; sibling to local `docs/`).

Source: [Turborepo installation](https://turborepo.dev/docs/getting-started/installation)

```bash
mkdir web && cd web
bunx create-turbo@latest .   # choose bun as package manager
git init && git add -A && git commit -m "chore: bootstrap turborepo monorepo"
```

Then, in order:

1. **Second app → `apps/test`.** After `create-turbo`, inspect `apps/`:
   - **If `apps/docs` exists** (default template): rename the folder to `apps/test`. Update that app's `package.json` `name`, all workspace references, and any `microfrontends.json` keys (`docs` → `test`). We do not ship a docs app in `web`; constitution lives in `bonafiai/docs`.
   - **If `apps/docs` does not exist:** add `apps/test` as a second Next.js app (mirror `apps/web` layout or follow Turborepo's add-app flow). Do not rename apps that are not there.
   - Either path must end with `apps/web` + `apps/test` before microfrontends wiring.
2. Remove `packages/eslint-config`. Replace with root oxlint + prettier per [Oxc guide](https://turborepo.dev/docs/guides/tools/oxc).
3. Add `packages/tailwind-config` per [Tailwind guide](https://turborepo.dev/docs/guides/tools/tailwind) (v4, `@theme` tokens).
4. Re-init `packages/ui` with `bunx shadcn@canary init` (monorepo option) per [shadcn guide](https://turborepo.dev/docs/guides/tools/shadcn-ui).
5. Add `packages/vitest-config` per [Vitest guide](https://turborepo.dev/docs/guides/tools/vitest).
6. Create root `lib/shared/`, `specs/`, `.specify/memory/`.
7. Commit after each major phase so diffs stay reviewable.

---

## Target shape

### Apps

```text
apps/
  web/                              product app, default in microfrontends
    app/
    proxy.ts                        edge session gate, this app only
    microfrontends.json             local: /test → test app
  test/                             internal test app
    app/
    proxy.ts                        own gate, own auth boundary
packages/
  ui/                               shadcn CLI installs here
    src/components/
  typescript-config/                @repo/typescript-config
  tailwind-config/                  @repo/tailwind-config
  vitest-config/                    @repo/vitest-config
lib/                                no modules yet; specs create them
  shared/                           events.ts, db.ts, inngest.ts, gateway.ts
specs/                              spec-kit artifacts
.specify/                           memory/constitution.md + templates
turbo.json
```

Locally, [microfrontends](https://turborepo.dev/docs/guides/microfrontends) composes both apps on one URL; production routing is separate. **`apps/web` is the full product Next.js app in production**, not a UI fragment. "Microfrontends" here means Turborepo/Vercel path routing between two deploys; vendor term only for config (`microfrontends.json`, dev proxy).

### App (per workspace)

```text
app/                    routes only, no business logic
  (app)/
    <feature>/
      page.tsx
      _components/
      _lib/
  api/
    [[...route]]/       one catch-all, Hono owns /api
    inngest/            Inngest serve handler (web only)
proxy.ts                at app root, gates this app only
```

Scaffold ships one route only: `app/(app)/home/page.tsx` with `return null;` in both apps. No `_components/`, no `_lib/` until features land.

### Lib (template, not scaffolded)

```text
lib/
  <module>/                         created only via approved spec
    router.ts                       Hono routes, mounted by app catch-all
    schema.ts                       Drizzle tables; types derived, never hand-written
    types.ts                        hand-written only when not derivable
    jobs/
      <job>.ts                      one file per job, thin Inngest shell; crons live here too
    <module>.ts                     domain logic, the fat part
  shared/
    events.ts                       zod event registry, noun.verb names
    db.ts                           tenant-scoped Drizzle client
    inngest.ts                      Inngest client
    gateway.ts                      shared infra only; grows slowly
```

### Rules

| Name | Description |
| --- | --- |
| Routes | `app/` routes only, no business logic |
| API | One catch-all per app; Hono routers live in `lib/<module>/router.ts` |
| Module shape | Every domain folder: `router.ts`, `schema.ts`, `types.ts`, `jobs/`, `<module>.ts` |
| Ownership | A module owns its tables, logic, jobs, and types |
| Boundaries | Modules talk through exported functions and events, never another module's tables |
| `shared/` | Events registry, tenant db, gateway client only; a type two modules need moves here, and that is a smell |
| UI | Route-private code in `_components` and `_lib`; shared components from `@repo/ui` |
| Specs | No feature work without `specs/NNN-*/spec.md`; no new `lib/<module>/` without an approved spec |

Types guardrails: derive from Drizzle (`$inferSelect`/`$inferInsert`) and zod (`z.infer`, `@hono/zod-validator`); API types via Hono RPC; hand-write only in `types.ts` when not derivable.

Jobs guardrails: one Inngest function per file in `jobs/`, thin shell, `noun.verb` events, one completion event, pick a stance (Recompute, Keyed, Re-runnable) in the spec.

---

## Shared config packages

### @repo/typescript-config

Source: [TypeScript guide](https://turborepo.dev/docs/guides/tools/typescript)

- `base.json`: `strict`, `noUncheckedIndexedAccess`, `module: NodeNext`, `target: es2022`
- `nextjs.json`, `react-library.json` extend base
- Each package has its own `tsconfig.json` extending the right preset; **no root tsconfig**
- No TypeScript Project References
- Node subpath `imports` (`#*`) over tsconfig `paths`

### @repo/tailwind-config

Source: [Tailwind guide](https://turborepo.dev/docs/guides/tools/tailwind)

- Tailwind v4, `shared-styles.css` with `@theme` tokens
- Export postcss config
- UI package uses `prefix(ui)` for component classes

### @repo/vitest-config

Source: [Vitest guide](https://turborepo.dev/docs/guides/tools/vitest)

- Shared config exported from `packages/vitest-config`
- Per-package `test` scripts: `vitest run`
- Per-package `test:watch`: `vitest --watch` (persistent, uncached in turbo)

### packages/ui

Source: [shadcn guide](https://turborepo.dev/docs/guides/tools/shadcn-ui)

```bash
cd apps/web
bunx shadcn@canary init   # monorepo option
```

Add components from inside an app; CLI installs to `packages/ui`.

---

## Multi-app routing (microfrontends config)

Source: [Microfrontends guide](https://turborepo.dev/docs/guides/microfrontends)

`apps/web` is the **product app**: full Next.js at `/` in production. `apps/test` is path-mounted at `/test`. Use vendor "microfrontends" only when editing routing config or linking Turborepo/Vercel docs.

`apps/web/microfrontends.json`:

```json
{
  "$schema": "https://turborepo.dev/microfrontends/schema.json",
  "options": { "localProxyPort": 3024 },
  "applications": {
    "web": {
      "packageName": "web",
      "development": { "local": { "port": 3000 } }
    },
    "test": {
      "packageName": "test",
      "development": { "local": { "port": 3001 } },
      "routing": [{ "paths": ["/test", "/test/:path*"] }]
    }
  }
}
```

Each app `package.json`:

```json
"dev": "next dev --port $(turbo get-mfe-port)"
```

`apps/test/next.config.ts`: `basePath: "/test"`.

No SPA `<Link>` across apps. Production: [Vercel microfrontends](https://vercel.com/docs/microfrontends).

---

## turbo.json tasks

| Task | Config | Source |
| --- | --- | --- |
| `//#lint` | Root oxlint | [Oxc](https://turborepo.dev/docs/guides/tools/oxc) |
| `//#format` | Root prettier or oxfmt | [Oxc](https://turborepo.dev/docs/guides/tools/oxc) |
| `check-types` | Per-package `tsc --noEmit`, `dependsOn: ["topo"]`, `topo` depends on `^topo` | [TypeScript](https://turborepo.dev/docs/guides/tools/typescript) |
| `test` | Per-package `vitest run` | [Vitest](https://turborepo.dev/docs/guides/tools/vitest) |
| `test:watch` | `cache: false`, `persistent: true` | [Vitest](https://turborepo.dev/docs/guides/tools/vitest) |
| `e2e` | Playwright packages, `dependsOn: ["^build"]`, `passThroughEnv: ["PLAYWRIGHT_*"]` | [Playwright](https://turborepo.dev/docs/guides/tools/playwright) |
| `dev` | Persistent, microfrontends proxy | [Microfrontends](https://turborepo.dev/docs/guides/microfrontends) |

Root `package.json` scripts:

```json
{
  "packageManager": "bun@1.2.0",
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo run lint",
    "format": "turbo run format",
    "check-types": "turbo run check-types",
    "test": "turbo run test",
    "test:watch": "turbo run test:watch",
    "e2e": "turbo run e2e",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate"
  }
}
```

---

## API (Hono)

Sources: [Hono best practices](https://hono.dev/docs/guides/best-practices), [Hono on Vercel](https://hono.dev/docs/getting-started/vercel)

Install in root or `apps/web`:

```bash
bun add hono @hono/zod-validator zod
```

`apps/web/app/api/[[...route]]/route.ts` (and same in `apps/test`):

- Mount a bare Hono app with `basePath('/api')`
- One inline `GET /health` returning `{ ok: true }` (no `lib/` router yet)
- Export `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD` via `handle(app)` from `hono/vercel`

Future modules: chained `new Hono().get(...)` in `lib/<module>/router.ts`, export `AppType`, mount with `app.route('/<module>', router)`.

---

## Infrastructure wiring (empty content)

### lib/shared/

```bash
bun add drizzle-orm @planetscale/database inngest zod
bun add -d drizzle-kit
```

- `events.ts`: empty zod event registry export
- `db.ts`: Drizzle client using `DATABASE_URL` from env (no tables yet)
- `inngest.ts`: Inngest client export
- `gateway.ts`: stub export

### Drizzle

Source: [Drizzle kit](https://orm.drizzle.team/docs/kit-overview)

Root `drizzle.config.ts` pointing at `lib/` schema glob (empty for now). Scripts `db:generate`, `db:migrate` in root `package.json`.

`.env.example`:

```bash
DATABASE_URL=postgresql://...
```

### Inngest

Source: [Inngest Next.js](https://www.inngest.com/docs/getting-started/nextjs-quick-start)

```bash
bun add inngest
```

`apps/web/app/api/inngest/route.ts`: serve handler wired to `lib/shared/inngest.ts`, **zero functions registered**.

`.env.example` add:

```bash
INNGEST_EVENT_KEY=
INNGEST_SIGNING_KEY=
```

### Smoke test

One trivial test in `lib/shared/` (e.g. `lib/shared/smoke.test.ts`) so `turbo run test` proves Vitest wiring.

---

## Auth (WorkOS AuthKit)

Source: [AuthKit Next.js](https://workos.com/docs/user-management/nextjs)

Both apps:

```bash
cd apps/web && bun add @workos-inc/authkit-nextjs
cd apps/test && bun add @workos-inc/authkit-nextjs
```

Per app:

- `proxy.ts`: `authkitMiddleware` in **middleware-auth mode** (all routes protected, allow-list exceptions)
- `app/layout.tsx`: wrap with `AuthKitProvider`
- `app/callback/route.ts`: AuthKit callback
- `app/login/route.ts`: server-side AuthKit authorization URL redirect
- **Google OAuth only** in WorkOS dashboard (disable email+password)
- Configure Sign-out redirect in dashboard

`.env.example` (per app or root):

```bash
WORKOS_API_KEY=
WORKOS_CLIENT_ID=
WORKOS_COOKIE_PASSWORD=   # openssl rand -base64 32, 32+ chars
NEXT_PUBLIC_WORKOS_REDIRECT_URI=http://localhost:3024/callback
```

FGA deferred.

---

## Dependencies

Install product deps as needed for wiring (not domain logic). Stub env placeholders for services without accounts yet.

### Product

| Name | Role | Doc |
| --- | --- | --- |
| Turborepo | Monorepo build, task runner, microfrontends | https://turborepo.dev/docs |
| Next.js 16 | App router, RSC, Vercel deploy | https://nextjs.org/docs |
| shadcn/ui | Shared UI in `packages/ui` | https://ui.shadcn.com/docs/monorepo |
| Hono | Typed API, catch-all, modular routers in `lib/` | https://hono.dev/docs |
| WorkOS | AuthKit sessions (`proxy.ts` gates) | https://workos.com/docs |
| Drizzle + PlanetScale | Typed SQL, migrations, dev branches | https://orm.drizzle.team/docs/overview |
| Zod | Validation + inference at boundaries | https://zod.dev |
| Inngest | Async jobs in `lib/<module>/jobs/` | https://www.inngest.com/docs |
| Vercel AI SDK | Agent loop in product | https://ai-sdk.dev/docs |
| Vercel AI Gateway | Model routing, BYOK | https://vercel.com/docs/ai-gateway |
| Pipedream Connect | Org-scoped connectors | https://pipedream.com/docs/connect |
| Cloudflare R2 | Object storage | https://developers.cloudflare.com/r2/ |
| LlamaParse | Parser seam | https://docs.cloud.llamaindex.ai/llamaparse/getting_started |
| Vercel | Hosting, preview deploys | https://vercel.com/docs |

Defer install until needed: Pipedream, LlamaParse, AI SDK (add when first agent feature spec lands).

### Tools

| Name | Role | Doc |
| --- | --- | --- |
| Bun | Runtime + package manager | https://bun.sh/docs |
| oxlint + prettier | Lint + format at repo root | https://turborepo.dev/docs/guides/tools/oxc |
| Vitest | Unit + integration | https://turborepo.dev/docs/guides/tools/vitest |
| Playwright | Acceptance / spec scenarios | https://turborepo.dev/docs/guides/tools/playwright |
| spec-kit | Spec-driven workflow | https://github.com/github/spec-kit |

---

## Agent workbench

Copy MCP config from the docs repo when scaffolding: `docs/.cursor/mcp.json` → `web/.cursor/mcp.json` (Mintlify + Linear).

### .cursor/mcp.json

| Server | Role |
| --- | --- |
| Mintlify | Read and search bonafiai/docs |
| Linear | Create, read, update issues |

### Skills

```bash
npx skills add https://mintlify.com/docs
```

Pin via `skills-lock.json`.

### spec-kit

Requires [uv](https://docs.astral.sh/uv/).

```bash
uv tool install specify-cli
specify init . --integration cursor-agent
```

Commands available in Cursor chat:

| Command | Role |
| --- | --- |
| `/speckit.constitution` | Write or update `.specify/memory/constitution.md` |
| `/speckit.specify` | Turn an issue into `spec.md` |
| `/speckit.clarify` | Resolve ambiguity before planning |
| `/speckit.plan` | Generate the implementation plan |
| `/speckit.checklist` | Quality-check the plan |
| `/speckit.tasks` | Break the plan into tasks |
| `/speckit.analyze` | Check spec, plan, and tasks agree |
| `/speckit.implement` | Execute the tasks |
| `/speckit.converge` | Confirm complete, queue gaps |

After spec-kit bump: `specify integration upgrade cursor-agent`.

### Root AGENTS.md contract

```markdown
- No feature work without `specs/NNN-*/spec.md`
- First agent action on a missing spec: `/speckit.specify`
- `.specify/memory/constitution.md` matches bonafiai/docs Architecture pages
- Agents file issues to Linear via MCP
- New `lib/<module>/` folders only via an approved spec
```

### Constitution seed

Run `/speckit.constitution` once scaffold is done. Transcribe Structure Rules + Types + Jobs from bonafiai/docs into `.specify/memory/constitution.md`.

---

## Repo hygiene

- `.gitignore`: `.turbo/`, `node_modules/`, `.env*.local`, `coverage/`, Next.js `.next/`
- `.env.example` committed (all keys above, no secrets)
- `README.md`: one paragraph + link to docs site
- CI: `.github/workflows/ci.yml` running `bun install && turbo run lint check-types test` ([GitHub Actions guide](https://turborepo.dev/docs/guides/ci-vendors/github-actions)); remote cache opt-in later
- Vercel: two projects (`web`, `test`); production microfrontends separate from local proxy

---

## Acceptance checks

Run in order when scaffold is complete:

1. `bun install` clean
2. `turbo dev` → `http://localhost:3024` serves web; `/test` serves test app
3. Unauthenticated requests redirect to AuthKit sign-in (Google)
4. `curl http://localhost:3024/api/health` returns `{"ok":true}`
5. `/home` renders (null) in both apps
6. `turbo run lint check-types test` green
7. `turbo run e2e` runs placeholder Playwright spec
8. `/speckit.constitution` available in Cursor; constitution seeded
9. `AGENTS.md` contract present; creating `lib/foo/` without a spec violates it

---

## Ready for the extraction pipeline

Foundation is done when a colleague opens Cursor, describes extraction stage 1 (source upload) as a Linear issue, and runs the spec-kit loop with zero setup questions.

First real specs are the pipeline stages: source upload → parse → structural extraction → … → commit. See pipeline notes in bonafiai/docs `.notes/`.

`lib/` holds only `shared/` until specs land.

---

## Explicit deferrals

All domain modules, Drizzle schema and migrations, Inngest functions (serve route ships empty), example UI components, WorkOS FGA, R2 buckets, Pipedream, LlamaParse, Linear issue template, coverage merging, `converge` CI gates beyond lint/check-types/test.
