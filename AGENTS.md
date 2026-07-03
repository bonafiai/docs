# Documentation project instructions

## About this project

This is the spec hub for Bonafi, the source of truth agents and humans build from. Code lives in [bonafiai/mono](https://github.com/bonafiai/mono). This repo holds beliefs, rules, workflow, testing doctrine, stack choices, and tooling rosters.

- Pages are MDX with YAML frontmatter
- Configuration lives in `docs.json`
- Install the Mintlify skill: `npx skills add https://mintlify.com/docs`

## Terminology

| Term | Meaning |
| --- | --- |
| Spec | A feature description with acceptance scenarios, what to build, not how |
| Rules | Tool-agnostic invariants every implementation satisfies (`concept/rules`) |
| Truth | Committed financial facts after human or agent approval |
| Tooling | Versioned agent workbench: MCP servers, skills, `AGENTS.md` |

## Writing standards

1. **Definition-first openers.** 1-2 sentences: what this is and why it matters. No throat-clearing.
2. **Numbered decomposition.** Break concepts into 2-4 named parts, one short paragraph each.
3. **Short catalogs.** Rosters as table rows or H3 sections with 1-2 sentence bodies.
4. **Tables.** Two columns where possible: Choice | Why. No "Rejected" columns.
5. **Callouts.** One per page max, genuine traps only.
6. **Card grids.** Navigation moments only (`index`, `quickstart`).
7. **No em dashes.** Use commas, periods, or restructure the sentence.
8. **1-2 minute read per page.** If a page takes longer, cut it.
9. **Definitive tone.** Everything documented is the current truth. No "might", "could", "we're considering".

General Mintlify rules:

- Second person, active voice, sentence case headings
- Every code block has a language tag
- Internal links: root-relative, no file extension (`/concept/workflow`)
- All images need descriptive alt text

## Content boundaries

Document: beliefs, rules, workflow, testing doctrine, stack overview, tooling rosters.

Do not document here: wiring implementation details, extraction pipeline stages, API reference, or open deliberations. Decisions happen in Slack and Linear, the docs state only the outcome.
