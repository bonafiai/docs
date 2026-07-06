---
name: Issues page for workflow
overview: Add method/issues.mdx, the structure of a Linear issue that feeds the spec-driven loop optimally, with the issue skeleton as its artifact, and wire Workflow's trigger and Describe step to it.
todos:
  - id: issues-page
    content: "Write method/issues.mdx: opener, skeleton artifact first, fields table, size line, inline spec-kit source"
    status: completed
  - id: layout-sweep
    content: Apply artifact-first + one-fact-one-place to jobs.mdx, types.mdx, structure.mdx; canon into AGENTS.md
    status: completed
  - id: workflow-wire
    content: "Workflow: opener and Describe link Issues, gherkin moves there, Tracking Capture row links it"
    status: completed
  - id: rewire-nav
    content: docs.json Method pages, quickstart Issues card, AGENTS.md symmetry and boundaries
    status: completed
  - id: ship
    content: mint validate, commit, push
    status: completed
isProject: false
---

# Issues: the artifact that starts the loop, plus the page layout canon

The loop starts with a Linear issue, and the issue seeds `spec.md`. It is the one artifact a human writes from scratch (and agents file them too, same rules), so it gets its own findable page under Method. Passes the page contract: doctrine text, a skeleton artifact, inline sources.

Target sidebar: Method becomes Workflow, Issues, Testing.

This plan also fixes the reading order across all pages and sets it as the canon:

**The page layout canon** (into AGENTS.md, applied to every page):

1. Opener, 1-2 sentences, the rule of the page.
2. Artifact immediately after, no scrolling: the tree, the skeleton, the code shape.
3. Tables and short sections after, explaining only what the artifact cannot say itself.
4. One fact, one place. A rule lives in the artifact's annotation, or a table row, or prose, never two of them. Whoever states it best keeps it.

All on `cursor/foundations-blueprint-9470`.

## 1. Write `method/issues.mdx`, artifact-first

Order per the canon: opener, skeleton, fields table, size line.

- **Opener**: an issue is the seed of a spec. The agent runs `/speckit.specify` from it, so the issue carries what and why in behavior terms, never the how. Applies to human-filed and agent-filed issues equally.
- **Skeleton immediately after the opener**, the artifact, no scrolling:

```md
Title: Reviewer approves proposed values

What: Reviewers approve or reject proposed values from parsed statements.
Uncertain mappings queue for review, approved proposals become truth.

Why: Truth must be human-approved, unreviewed values cannot enter reports.

Scenarios:
- Given a parsed statement, when the reviewer approves the ending balance
  proposal, then truth contains the committed metric for that investor.
- Given an uncertain mapping, when parsing completes, then it queues for review.

Not now: bulk approval, delegation.
```

- **Fields table below it**, each row saying only why the field exists, never restating the skeleton's content:

| Name | Description |
| --- | --- |
| Title | The behavior, not the implementation |
| What | Behavior terms force the agent to plan the how itself |
| Why | The value, it becomes the spec's justification |
| Scenarios | Given/when/then seeds, they become the spec's acceptance scenarios |
| Not now | Explicit non-scope, the guard against agent over-eagerness |

- **Size line**: a full skeleton is for features (the full loop). Fixes need title plus one regression line, the light path.
- Inline source on `/speckit.specify` pointing at [spec-kit](https://github.com/github/spec-kit) (its guidance: explicit what and why, no tech stack at this stage).

## 2. Layout sweep on the existing pages

Apply the canon and the one-fact-one-place rule:

- [architecture/jobs.mdx](architecture/jobs.mdx): the Shape skeleton moves up, directly after the opener. The anatomy table follows it, the Stances table last. Dedupe: `noun.verb` naming currently lives only in the opener, keep it there; "thin shell" is in both the opener and the anatomy table's Shell row, the opener keeps it, the Shell row becomes "Delegates to the domain module, no logic in the shell".
- [architecture/types.mdx](architecture/types.mdx): the code artifact moves up, directly after the opener. The five-row table follows, the shared-smell line closes. Dedupe check: table rows and code comments say different things already, keep.
- [architecture/structure.mdx](architecture/structure.mdx): tree is already first, correct. Dedupe: the jobs sentence in the Modules section (one file per job, thin shell, cron trigger) duplicates [architecture/jobs.mdx](architecture/jobs.mdx)'s opener, replace with "its jobs live in `jobs/`, see [Jobs](/architecture/jobs)". The tree annotation stays.

## 3. Wire Workflow to it

In [method/workflow.mdx](method/workflow.mdx):

- Opener: "The loop starts the moment a [Linear issue](/method/issues) is handed to an agent in Cursor."
- Describe step slims, no duplication: "Hand the agent the issue, written per [Issues](/method/issues). The agent runs `/speckit.specify` and returns `spec.md` for your review." The gherkin example moves to Issues (it is the Scenarios field's shape), Describe keeps only the scenario-id-binds-to-test line and the success line.
- Tracking's Capture row links Issues: agent-filed issues follow the same structure.

## 4. Rewire

- [docs.json](docs.json): Method pages `["method/workflow", "method/issues", "method/testing"]`.
- [quickstart.mdx](quickstart.mdx): add an Issues card ("The issue structure that starts the loop") to the Explore grid.
- [AGENTS.md](AGENTS.md): symmetry table Method row becomes "Workflow, Issues, Testing"; content boundaries gains "issues"; the layout canon lands in Writing standards (artifact right after the opener, tables explain only what the artifact cannot, one fact one place).

## 5. Ship

`npx mint validate`, commit, push.