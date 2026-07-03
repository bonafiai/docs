# Documentation project instructions

## About this project

This is the **spec hub** for Bonafi — the source of truth agents and humans build from. Code lives in [bonafiai/mono](https://github.com/bonafiai/mono); this repo holds beliefs, rules, workflow, stack choices, and decisions.

- Pages are MDX with YAML frontmatter
- Configuration lives in `docs.json`
- Install the Mintlify skill: `npx skills add https://mintlify.com/docs`

## Terminology

| Term | Meaning |
| --- | --- |
| Spec | A feature description with acceptance scenarios — what to build, not how |
| Rules | Tool-agnostic invariants every implementation must satisfy (`concept/rules`) |
| Truth | Committed financial facts after human or agent approval |
| Tooling | Versioned agent workbench: MCP servers, skills, `AGENTS.md` |

## Writing standards

Follow the Cursor-docs-inspired contract:

1. **Definition-first openers** — 1–2 sentences: what this is and why it matters. No throat-clearing.
2. **Numbered decomposition** — Break concepts into 2–4 named parts, one short paragraph each.
3. **Short catalogs** — Rosters as table rows or H3 sections with 1–2 sentence bodies.
4. **Accordions** — Optional depth on overview pages (`AccordionGroup`).
5. **Tables** — Enumerable comparisons only (choice / why / rejected).
6. **Callouts** — One per page max, genuine traps only.
7. **Card grids** — Navigation moments only (`index`, `quickstart`).

General Mintlify rules:

- Second person, active voice, sentence case headings
- Every code block has a language tag
- Internal links: root-relative, no file extension (`/concept/workflow`)
- All images need descriptive alt text

## Page template

Each page: purpose paragraph → flowchart or table (if applicable) → hard rules or open decisions (if applicable). Stay within page budgets in the plan.

## Content boundaries

Document: beliefs, rules, workflow, testing doctrine, decisions, stack overview, tooling rosters.

Do not document here: wiring implementation details, extraction pipeline stages, or API reference — those flow in later from the product core.
