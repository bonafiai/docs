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
| Guardrails | Our product-specific invariants, stated per architecture page, transcribed into the code repo's constitution |
| Truth | Committed financial facts after human or agent approval |
| Tooling | Versioned agent workbench: MCP servers, skills, `AGENTS.md` |

## Writing standards

1. **Definition-first openers.** 1-2 sentences: what this is and why it matters. No throat-clearing.
2. **Scannable above all.** The 1-2 minute budget is for the scan with accordions collapsed, not the word count. Depth folds into accordions and tables instead of being cut. Rules and checklists never go inside an accordion.
3. **Command-first.** Where a page has commands, they come before the explanation.
4. **Tables.** Two columns: Name | Description. No "Rejected" columns, no status theater.
5. **Headings one word** where possible, sentence case.
6. **No em dashes.** Use commas, periods, or restructure the sentence.
7. **Definitive tone.** Everything documented is the current truth. No "might", no open deliberations, decisions happen in Slack and Linear.
8. **Callouts.** One per page max, genuine traps only.
9. **Card grids.** Navigation moments only (`index`, `quickstart`).
10. **Cut process narration, keep doctrine.** Cut what tools do on their own. Keep explicit what a colleague would otherwise do wrong: derived types, placement, boundaries.
11. **Source, don't replicate.** External tool detail lives in the tool's own docs, linked once. Our pages carry only what is ours: decisions, touchpoints, house rules. Replicated docs go stale, links don't.

## Symmetry

Page names orient, pages act. Every page concise, never a mega-page. One-word page names, plain nouns, no metaphors in the sidebar.

| Group | Pages |
| --- | --- |
| Getting started | Introduction, Quickstart |
| Method | Workflow, Testing |
| Architecture | Structure, Types, Jobs, more as they land |
| Stack | Overview, Tooling |

The page contract: one concern per page, findable by name. A page ships text, sources, and an artifact (tree, table, or code skeleton), or it does not exist yet.

## Diagrams

- Vertical `TD` for chains, 9 nodes max per diagram.
- Linear flows are tables or Steps, not wide LR diagrams.
- Keep diagrams tall enough for Mintlify zoom controls.

General Mintlify rules:

- Second person, active voice, sentence case headings
- Every code block has a language tag
- Internal links: root-relative, no file extension (`/method/workflow`)
- All images need descriptive alt text

## Content boundaries

Document: workflow, testing, architecture (structure, types, jobs, more as they land), stack, tooling.

Do not document here: per-SDK implementation depth (comes later as its own pass), API reference, or open deliberations.

Roster table Names link to the exact official docs of the tool.
