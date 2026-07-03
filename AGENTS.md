# Documentation project instructions

## About this project

This is the spec hub for Bonafi, the source of truth agents and humans build from. Code lives in [bonafiai/mono](https://github.com/bonafiai/mono). This repo holds the workflow, testing doctrine, stack choices, and tooling rosters.

- Pages are MDX with YAML frontmatter
- Configuration lives in `docs.json`
- Install the Mintlify skill: `npx skills add https://mintlify.com/docs`

## Terminology

| Name | Description |
| --- | --- |
| Spec | A feature description with acceptance scenarios, what to build, not how |
| Guardrails | Our product-specific invariants (`concept/workflow#guardrails`) |
| Truth | Committed financial facts after human or agent approval |
| Tooling | Versioned agent workbench: MCP servers, skills, `AGENTS.md` |

## Writing standards

1. **Definition-first openers.** 1-2 sentences: what this is and why it matters. No throat-clearing.
2. **Scannable above all.** 1-2 minute read per page. If it takes longer, cut it.
3. **Command-first.** Where a page has commands, they come before the explanation.
4. **Tables.** Two columns: Name | Description. No "Rejected" columns, no status theater.
5. **Headings one word** where possible, sentence case.
6. **No em dashes.** Use commas, periods, or restructure the sentence.
7. **Definitive tone.** Everything documented is the current truth. No "might", no open deliberations, decisions happen in Slack and Linear.
8. **Callouts.** One per page max, genuine traps only.
9. **Card grids.** Navigation moments only (`index`, `quickstart`).

General Mintlify rules:

- Second person, active voice, sentence case headings
- Every code block has a language tag
- Internal links: root-relative, no file extension (`/concept/workflow`)
- All images need descriptive alt text

## Content boundaries

Document: workflow, testing, stack, tooling, and later the architecture pages.

Do not document here: wiring implementation details (until the Architecture pages land), API reference, or open deliberations.
