---
name: Fold commands, slim pages
overview: Fold the Commands page back into Workflow as a compact touchpoints section, drop the tall diagram from the architecture Overview, and rework Structure so the placeholder tree carries the placement answers with fewer, more flowing sections.
todos:
  - id: fold-commands
    content: Delete method/commands.mdx, fold touchpoints table into workflow.mdx Loop, rewire docs.json, quickstart, stack/overview
    status: completed
  - id: slim-overview
    content: Remove the tall diagram from architecture/overview.mdx, keep opener + planes table + closing line
    status: completed
  - id: rework-structure
    content: "Structure: placeholder tree absorbs Placement table, merge into Modules/Types/App/Boundaries with Types kept explicit (derive, don't hand-write)"
    status: completed
  - id: agents-ship
    content: AGENTS.md symmetry + boundaries, mint validate, commit, push
    status: completed
isProject: false
---

# Revision: fold Commands, slim Overview, rework Structure

Three corrections from review, all on `cursor/foundations-blueprint-9470`.

## 1. Fold Commands back into Workflow

The distinction that decides it: [method/testing.mdx](method/testing.mdx) documents commands you run, so it stays a page. [method/commands.mdx](method/commands.mdx) documents commands the agent runs, so it compresses to what the human actually needs and moves into Workflow.

- Delete [method/commands.mdx](method/commands.mdx). Method is a one-pager (Workflow) plus Testing.
- In [method/workflow.mdx](method/workflow.mdx), the Loop section becomes: the diagram, then one short paragraph ("The agent runs every `/speckit.*` command. Your job is three touchpoints."), then one compact table:

| Name | Description |
| --- | --- |
| Describe | What and why in behavior terms, no tech stack. The spec's scenarios must cover what you meant |
| Answer | The clarify questions, and review the plan: agents are over-eager, catching invented components is your job |
| Review | The PR with every scenario test green, converge leftovers became Linear issues |

That preserves the only genuinely human content from the Commands page in eight lines.

- The Loop section closes with the sourcing line, stated once: command reference, artifacts, and process detail live in [spec-kit](https://github.com/github/spec-kit), we deliberately do not replicate them, so nothing here goes stale when the tool updates. This is the pattern for every external tool: our pages carry what is ours (decisions, touchpoints, house rules), the source repo carries the rest.

- [docs.json](docs.json): Method back to `["method/workflow", "method/testing"]`.
- [quickstart.mdx](quickstart.mdx): remove the Commands card.
- [stack/overview.mdx](stack/overview.mdx): spec-kit row points back at `/method/workflow`.

## 2. Slim the architecture Overview

The 11-node vertical diagram fills the whole page, and it violates our own convention (linear flows are tables, not diagrams). Remove it. The page becomes: two-line opener, the planes table, the closing kitchen line. The table already says everything the diagram said.

## 3. Rework Structure: the tree is the recipe

In [architecture/structure.mdx](architecture/structure.mdx):

- The tree switches to placeholder naming (`<module>`, `<feature>`) and absorbs the Placement table's answers into its annotations, so it reads as the idempotent shape you follow for every module:

```text
app/                    routes only, no business logic
  (app)/                route group, the authenticated product
    <feature>/
      page.tsx
      _components/      UI private to this route
      _lib/             helpers private to this route
  api/
    [[...route]]/       one catch-all, Hono owns /api
lib/
  <module>/             one folder per domain, same shape every time
    router.ts           its Hono routes, mounted by the catch-all
    schema.ts           its Drizzle tables, types derived, not written
    types.ts            hand-written domain types, only if not derivable
    jobs/               its Inngest functions, one file per job, crons included
    <domain>.ts         its domain logic, the fat part
  shared/               events registry, tenant db, gateway client
proxy.ts                the edge session gate
specs/                  feature specs from the loop
.specify/               spec-kit memory, the constitution
```

- Delete the Placement table (absorbed above).
- Merge the five H2 sections into four, written as connected prose instead of fragments. The editorial rule for what survives the merge: process narration gets cut, doctrine that changes how a colleague works stays explicit.
  - **Modules**: ownership as one narrative: a domain folder owns its logic, router, tables, jobs, and types, per [Hono best practices](https://hono.dev/docs/guides/best-practices); jobs fold in here (one file per job in `jobs/`, thin shell delegating to domain logic, a cron is just a job with a [cron trigger](https://www.inngest.com/docs/guides/scheduled-functions)).
  - **Types**: stays its own explicit section, not merged away. This is the doctrine a new colleague breaks first: you do not hand-write types or schemas, you derive them. Opens with that rule, then a Name | Description table making each source explicit:

| Name | Description |
| --- | --- |
| Table types | Derived from Drizzle, `$inferSelect` and `$inferInsert` in `schema.ts`, never written by hand |
| Event types | Derived from the zod registry via `z.infer` |
| API types | Flow to the client through Hono RPC, no hand-written DTOs |
| Domain types | The only hand-written ones, in `types.ts`, only when not derivable |

  Doc links kept: [Drizzle schema declaration](https://orm.drizzle.team/docs/sql-schema-declaration), [zod inference](https://zod.dev/?id=type-inference), [Hono RPC](https://hono.dev/docs/guides/rpc).
  - **App**: routing only, route groups, private folders, the catch-all handing `/api` to Hono.
  - **Boundaries**: modules talk through functions and events, never tables; `shared/` has three residents and growth means something is misplaced; a type two modules need moves there, and that is a smell.

**Growth path, agreed but not built now**: each section is the seed of its own Architecture page, graduating when it outgrows a paragraph and the codebase exists to anchor it. Jobs is the first candidate: the canonical, idempotent job shape (one trigger, concurrency key, idempotency stance, one completion event, the content the deleted events page carried) becomes `architecture/jobs.mdx` the moment the first real Inngest function lands. Same control the tree gives over folders, applied per plane.

## 4. Update AGENTS.md and ship

- Symmetry table: Method row back to Workflow | Testing.
- Content boundaries: drop "commands".
- Add the editorial balance rule to Writing standards, it is the principle behind every cut and keep in this pass: "Cut process narration (what tools do on their own). Keep doctrine explicit (what a colleague would otherwise do wrong: derived types, placement, boundaries)."
- Add the sourcing rule next to it: "Source, don't replicate. External tool detail lives in the tool's own docs, linked once. Our pages carry only what is ours: decisions, touchpoints, house rules. Replicated docs go stale, links don't."
- Add the growth rule to the symmetry section: "A section graduates to its own page when it outgrows a paragraph and the codebase exists to anchor it. Jobs is next."
- `npx mint validate`, commit, push.