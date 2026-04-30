---
description: Route freeform intent to the right Mendix/MDL workflow
updated: 2026-04-29
tags: [mendix, mxcli, mdl]
---

<role>
Mendix workflow router. Parse user intent. Load the `mendix` skill and relevant sub-skill. Dispatch to the appropriate workflow file.
</role>

<arguments>
`/mendix <intent> [module] [entity|microflow|page]`
</arguments>

<context>
Read the active codebase file (`codebases/<project>.md`) for: working directory, project file name, and module conventions.
Binary: `./mxcli` — NEVER bare `mxcli`
App runtime: Docker (`./mxcli docker run -p <project>.mpr --wait`)
Always load the `mendix` skill first, then the sub-skill matched below.
</context>

<workflow_dispatch>
More specific pattern wins. Ties → present options.

| Intent pattern | Workflow |
|----------------|----------|
| "explore / show structure / what's in / describe element" | `mendix-explore` |
| "refresh catalog / rebuild catalog" | `mendix-refresh-catalog` |
| "check script / validate MDL / syntax check" | `mendix-check-script` |
| "validate project / run mx check / CE errors" | `mendix-validate-project` |
| "lint / check quality / find issues / report / score / best practices" | `mendix-lint` |
| "diff script / what would change / preview changes" | `mendix-diff-script` |
| "diff local / show git changes / uncommitted changes" | `mendix-diff-local` |
| "create entity / add entity / new entity / add attribute / add association" | `mendix-create-entity` |
| "create CRUD / generate CRUD / overview + edit pages / scaffold" | `mendix-create-crud` |
| "create microflow / write microflow / create nanoflow / create page / create snippet / create security / create navigation / REST / OQL / any other write" | `mendix-write` |
| "test / run tests / verify app / check page / check data" | `mendix-test` |
| "docker / run app / start app / stop app / restart / logs / reload" | load `mendix` skill — see `<docker>` section |
</workflow_dispatch>

<steps>

<step n="1" name="Load mendix skill">
Load `mendix` skill for core conventions and rules.
<done_when>Skill loaded.</done_when>
</step>

<step n="2" name="Parse arguments">
Parse `$ARGUMENTS`: action verb, subject, module, element name.
<done_when>Components extracted.</done_when>
</step>

<step n="3" name="Match and dispatch">
Match intent against workflow dispatch table.
- Single match: announce selection and load the workflow.
- 2–3 plausible matches: present options, await confirmation.
- No match: ask one clarifying question.

Workflow files in `.opencode/workflows/`:
- `mendix-explore` — structure, describe, catalog queries, cross-reference
- `mendix-refresh-catalog` — rebuild catalog database
- `mendix-check-script` — MDL syntax + reference validation
- `mendix-validate-project` — full mx check (CE errors)
- `mendix-lint` — lint rules + scored report
- `mendix-diff-script` — preview script changes against project
- `mendix-diff-local` — show uncommitted git changes
- `mendix-create-entity` — create entity with attributes and associations
- `mendix-create-crud` — full CRUD scaffold (entity + overview + NewEdit pages)
- `mendix-write` — create/modify any other element (microflows, pages, security, navigation, etc.)
- `mendix-test` — Docker start, playwright UI verification, OQL data assertions
<done_when>Workflow dispatched and executing.</done_when>
</step>

</steps>

<output_rules>
- Language: English.
- Never show raw MDL unless user explicitly asks.
- Always quote identifiers in MDL scripts.
- Maximum one clarifying question before proceeding.
- Report outcomes as plain language summaries.
</output_rules>

$ARGUMENTS
