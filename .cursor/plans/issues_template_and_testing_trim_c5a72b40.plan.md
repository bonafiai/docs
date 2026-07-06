---
name: Issues template and testing trim
overview: Turn method/issues into a doing-first page led by a copyable issue template, trim method/testing to the doings, make method/workflow show why /speckit.specify is non-optional while folding the constitution to a single home, add a lean Customization note to Tooling (extensions/presets/overrides, none adopted), and note our DDD structure reaches spec-kit via the constitution gate plus project-local overrides.
todos:
  - id: issues-template
    content: "Rework method/issues.mdx: pipeline-anchored opener, copyable issue template immediately after (fields encode guardrails), drop standalone guardrails table, keep tight Kinds + Lifecycle tables, note canonical format/Linear template, handoff clarifying only features hit /speckit.specify"
    status: completed
  - id: workflow-enforce
    content: "Update method/workflow.mdx: Describe step shows /speckit.specify is enforced (agent contract in Tooling + spec-reference gate), fold constitution into the Answer step (link Architecture), remove the standalone Constitution section; add two lines to architecture/structure.mdx: constitution gate + proactive DDD layout via project-local overrides/preset"
    status: completed
  - id: testing-trim
    content: "Trim method/testing.mdx: relay opener, keep Commands (fold acceptance-proves-spec into Playwright row), delete Layers table + standalone acceptance paragraph, compress Gate to the required commands set, keep Warning"
    status: completed
  - id: tooling-customization
    content: "Add a lean ## Customization note to stack/tooling.mdx: extensions add capabilities, presets/overrides change how spec-kit generates artifacts, we adopt none by default and add via PR, DDD layout via overrides (-> Structure), resolution top-wins. No roster, no specific extension."
    status: completed
  - id: verify
    content: Run mint validate; confirm no docs.json/AGENTS.md/quickstart changes needed
    status: completed
isProject: false
---

## Goal (kept in focus)

The `Method` pages lean toward doing while keeping the spec-driven-truth thread visible and, crucially, showing what enforces it (not just asserting it). The issue is the start of the pipeline and the shared template is the only lever for issue symmetry upstream; `/speckit.specify` is not optional because the agent contract compels it and the `spec-reference` gate blocks the alternative; the constitution binds every plan and lives in one home. Tooling gains a lean Customization note (extensions/presets/overrides) so the spec-kit customization surface is documented and governed, though we adopt no extension by default. No new pages, no nav/`docs.json`/`AGENTS.md` changes.

Decision folded in: `spec-kit-linear-sync` was evaluated and dropped. Its value is cross-repo visibility, and we are one mono repo, so it is not worth the unreviewed community code, API key, git hooks, and Action. The issue stays a plain Linear issue the agent keeps current, no mirror.

## 1. Rework [method/issues.mdx](/Users/paddy/www/docs/method/issues.mdx)

Make it artifact-first: a copyable template is the "thing to do", the guardrails live inside it as fields instead of a separate table to read.

- Re-anchor the opener to the pipeline and keep the "one format" thesis: an issue is where spec-driven work starts; anyone files one, human or agent; it must let any agent pick it up cold; copy this and fill every field.
- Add the template snippet immediately after the opener (renders with a copy button in Mintlify). This replaces the current standalone guardrails table:

```md
Title: <behavior, not implementation>
Kind: feature | fix | chore

## Intent
What: <the behavior, no tech stack>
Why: <the value it delivers>

## Context
- <docs page or official source>
- <related issue or PR>
- <originating chat>

## Scope
Not now: <what this explicitly excludes>
```

- One line under it for the two non-obvious rules: intent in behavior terms (the how is the agent's job), and every link a cold reader would otherwise hunt for.
- One line locating the format (the "defined file" you want): this is the canonical format, and it lands as the Linear issue template so new issues open pre-filled. Wiring it into Linear/the v2 repo is out of scope for the docs repo (decision: docs holds the canonical text now).
- Keep the `Kinds` table (Feature/Fix/Chore), still linking to [Workflow](/method/workflow) for the path so the path definition stays in one place.
- Keep a tight `Lifecycle` table (Filed/Worked/Closed), preserving the "worked = post decisions mid-work" rule (the agent keeps the issue current by hand, no auto-mirror).
- Handoff clarifies where the hard check is and points to Testing: only features enter spec-kit via [`/speckit.specify`](https://github.com/github/spec-kit); done is the gate, see [Testing](/method/testing).

Net: opener + copyable template + 2 short tables + handoff. One clear action (copy, fill), guardrails embedded, less prose.

## 2. Trim [method/testing.mdx](/Users/paddy/www/docs/method/testing.mdx)

Cut to the doings, remove the duplication that the `Layers` table creates with `Commands`.

- Opener carries the relay line so it completes the handoff from Issues: a feature is done when its scenario tests pass, not when the agent says so; run these locally, CI runs the same.
- Keep the `Commands` table (the doing). Fold the one useful acceptance point into the Playwright row: "Acceptance, one test per spec scenario, the proof the agent built the behavior."
- Delete the `Layers` table and the standalone acceptance paragraph. Every layer is already implied by a command (lint = static + invariant, typecheck = static, test = unit + integration, playwright = acceptance), so the table is pure duplication.
- Compress `Gate` to the honest set: the same commands run as required checks, all deterministic and key-free; LLM evals run nightly, never in the gate; docs-only PRs use a companion workflow. Removes the mislisted six-check list (`spec-reference` is inside `lint`, not a separate check).
- Keep the `<Warning>` that the gate does not grow without a written reason (genuine guardrail).

Net: opener + Commands table + push line + short Gate + Warning. Notably smaller, all actionable.

## 3. Fix [method/workflow.mdx](/Users/paddy/www/docs/method/workflow.mdx) enforcement + fold the constitution

Stop asserting, start showing what makes the loop reliable, and remove the 3-way constitution duplication (currently in Workflow, the Structure tree, and the Tooling contract).

- Opener: strip the meta sentence "We do not replicate the tool's docs, this page is only what you act on." (throat-clearing, not actionable). The preceding sentence already establishes the loop and the three touchpoints.
- `Describe` step: make `/speckit.specify` non-optional and cite both mechanisms, without re-explaining them. New text: it is the agent's first move because the [contract](/stack/tooling) requires it and the `spec-reference` gate blocks a feature branch without a `spec.md`, then review the returned spec (scenarios cover what you meant, nothing invented, each binds to one test id like `CAP-001`).
- `Answer` step: fold the constitution in where the plan is actually checked, "review the plan against the [constitution](/architecture/structure)".
- Remove the standalone `## Constitution` section (lines 26-28). Its two useful facts relocate: "agent checks the plan against it" is now in the `Answer` step; "CI enforces the mechanical subset" moves to Structure (below).
- The closing bridge line and `## Path` stay.

Constitution home in [architecture/structure.mdx](/Users/paddy/www/docs/architecture/structure.mdx): add two short lines after the tree so Architecture is the single content home.
- Gate line: these pages are the constitution in human form, transcribed into `.specify/memory/constitution.md`, the agent gates every plan against it and CI enforces the mechanical subset, see [Testing](/method/testing).
- Proactive line (lean presets answer): name this as our domain-driven layout and note it also reaches spec-kit proactively, the same conventions and terminology shape spec-kit's plan/tasks templates via project-local overrides in `.specify/templates/overrides/` (a shareable preset only if we go multi-repo), so generated artifacts are born in the `lib/<module>/` shape, not merely checked against it.

The tree's `.specify/` line stays; the Tooling `## Contract` line stays (it is the agent's rulebook pointing here).

Net: Workflow shrinks to opener + 3 steps + Path + bridge, and reads as "why this is reliable" not "trust me". Constitution explained once, in Architecture.

## 4. Add a Customization note to [stack/tooling.mdx](/Users/paddy/www/docs/stack/tooling.mdx)

No extension is adopted, so instead of a roster add a short `## Customization` section (after `## Dependencies`, before `## Contract`, grouping the spec-kit governance) that documents the surface and how we govern it.

- Three sentences, no table: spec-kit is tailored three ways, extensions add capabilities (`specify extension add`), while presets and project-local overrides in `.specify/templates/overrides/` change how it generates specs/plans/tasks (our DDD layout and terminology); resolution is top-wins (`overrides > presets > extensions > core`).
- Governance and truth: we adopt none by default and add deliberately via PR, same as servers/skills/deps; the [community catalog](https://github.github.io/spec-kit/community/extensions.html) is unreviewed, so we read the source and pin a release before adopting. Point to [Structure](/architecture/structure) as where the DDD convention the overrides express actually lives.
- No frontmatter/description or roster changes: this is a note, not a fourth roster.

Net: the spec-kit customization surface is documented and governed without claiming an adoption we did not make; the DDD-via-overrides fact stays anchored in Structure with Tooling pointing to it.

## 5. Verify

- `mint validate` (or the repo's validate script) to confirm MDX/frontmatter parse.
- No `docs.json`, `AGENTS.md`, or `quickstart.mdx` edits: no pages added/renamed, and existing cards still describe the reworked pages.

## Out of scope

- Wiring the template into Linear or a v2 repo `ISSUE_TEMPLATE`.
- Adopting any spec-kit extension. `spec-kit-linear-sync` was evaluated and dropped (cross-repo tool, we are one mono repo); the Customization note documents the surface without adopting anything.
- A standalone Constitution page (folded per decision; content home is Architecture).
- New pages or nav changes; edits touch five existing pages only (`issues`, `testing`, `workflow`, `structure`, `tooling`).