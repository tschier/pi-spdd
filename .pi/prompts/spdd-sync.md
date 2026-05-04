---
description: Sync accepted code changes back into an SPDD Canvas prompt
argument-hint: "<@spdd/prompt/file.md or path> [changed files, git diff context, or description]"
---

# SPDD Sync

Synchronize implementation details from changed code back into a saved SPDD structured prompt using the **REASONS Canvas** framework:

- **R** — Requirements
- **E** — Entities
- **A** — Approach
- **S** — Structure
- **O** — Operations
- **N** — Norms
- **S** — Safeguards

This is the reverse-flow phase of the SPDD workflow. It updates the prompt so it accurately reflects the current implementation after code review, refactoring, cleanup, bug-fix implementation, or other code-side changes that the team has accepted.

Source inspiration:

- OpenSPDD `/spdd-sync` command: https://github.com/gszhangwei/open-spdd/blob/v0.4.9/internal/templates/data/core/spdd-sync.md
- Martin Fowler, "Structured-Prompt-Driven Development (SPDD)": https://martinfowler.com/articles/structured-prompt-driven/

Core SPDD principle:

> The structured prompt should reflect the actual implementation, not just the planned implementation.

## Input

The user's input is:

```text
$ARGUMENTS
```

Input must include a saved REASONS Canvas prompt, usually under:

```text
spdd/prompt/
```

Input may also include changed file paths, a description of the refactor/change, or instructions about what should be synchronized.

Examples:

```text
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md src/users/user-service.ts src/users/user-service.test.ts
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md sync the renamed validator and service extraction from the latest git diff
```

Note on Pi references: in Pi, `@file` is usually a UI-level file attachment. If file content is already attached or inlined in the conversation, treat it as already read; do not re-fetch it unless necessary to resolve ambiguity.

## Phase Goal

Update the saved structured prompt so its Entities, Approach, Structure, Operations, Norms, and Safeguards match the accepted current implementation.

The prompt remains the version-controlled specification for future work. `/spdd-sync` keeps that specification truthful after code-side evolution.

## Phase Boundary

`/spdd-sync` owns only the Code → Canvas synchronization phase.

It may:

- read a saved REASONS Canvas prompt
- inspect changed or relevant source files
- compare actual implementation against the prompt
- propose prompt updates
- update the prompt after explicit user approval
- validate prompt internal consistency after the update

It must not:

- generate or modify implementation code
- invent new requirements that are not supported by accepted code or explicit user instruction
- relax business requirements, safeguards, security constraints, or acceptance criteria silently
- inline `/spdd-generate` or `/spdd-prompt-update` behavior
- create commits unless the user explicitly asks

If the code change represents a new business requirement or a desired behavior/design change not already accepted in code, stop and instruct the user to run `/spdd-prompt-update <prompt-file> <change>` or ask the user to explicitly approve the Canvas change. If the prompt is updated and code needs regeneration afterward, instruct the user to run `/spdd-generate <prompt-file>`. Pi does not auto-chain prompt templates.

## Steps

### 1. Validate and consolidate input

If no input is provided, stop and ask the user:

> Please provide the path to the structured prompt file you want to sync, for example `@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md`. You may also include changed file paths or a brief description of what changed.

Do **not** proceed without a valid prompt file path.

Resolve the prompt file:

- Treat already attached or inlined `@file` content as read.
- If an argument starts with `@` but no file is attached or inlined, strip the leading `@` and treat the remainder as a plain path.
- If a plain path points to a prompt file, use `read` to read the entire file.
- If a glob is provided for the prompt file, use `bash` to resolve it. Continue only if it resolves to exactly one prompt file; otherwise ask the user to choose.
- If the prompt file cannot be resolved or read, report the problem and ask for a valid path.

Resolve sync context:

- Treat additional existing paths or `@file` references as changed/relevant files to inspect.
- Treat additional text as user-provided change context.
- If no changed files or change description is provided, inspect repository-local git state with `bash` commands such as `git status --short`, `git diff --name-only`, and focused `git diff -- <path>` where available.
- If the repository is not a git repository or no changes can be identified, ask the user which components changed before proceeding.

Before continuing, verify that:

- the full structured prompt has been read
- every user-specified changed file has been read or an error was surfaced
- the sync scope is clear enough to compare code and prompt

### 2. Parse the REASONS Canvas

Identify the existing prompt sections and their sync priority:

| Section | Purpose | Sync priority |
|---------|---------|---------------|
| **Requirements** | Overall goal, scope, DoD, AC/DE IDs | Out of scope by default — change only with explicit approval; route requirement/design changes through `/spdd-prompt-update <prompt-file> <change>` |
| **Entities** | Domain model, data shapes, relationships | High — class/type/data relationships may change |
| **Approach** | Implementation strategy and trade-offs | Medium — architectural decisions or patterns may evolve |
| **Structure** | Components, files, dependencies, data flow | High — files, dependencies, layers, and relationships may change |
| **Operations** | Concrete implementation tasks | Highest — implementation details most often drift |
| **Norms** | Engineering standards and project conventions | Medium — reusable patterns may emerge |
| **Safeguards** | Non-negotiable constraints | Low — update only when code truly changed and change is acceptable |

If the prompt is missing required sections or is too malformed to update safely, stop and report the issue before editing.

### 3. Identify affected components

Determine which components were changed or should be synced.

Sources, in priority order:

1. User-specified files/classes/components in `$ARGUMENTS`
2. User-provided change description
3. Git working tree changes, staged changes, or recent diff context
4. File paths and components listed in the Canvas Structure and Operations sections

For each affected component, capture:

- file path
- component/class/function/module name
- type of change: renamed, moved, restructured, logic changed, validation changed, dependency changed, new component, deleted component, tests changed
- brief description of observed change
- relevant AC/DE tags if present in the prompt

If affected components are ambiguous, stop and ask the user to clarify:

```markdown
I found multiple possible sync scopes. Which components should be synchronized?

- [candidate path/component]
- [candidate path/component]

Please specify the files/classes and the kind of change to sync.

Pi does not auto-chain prompt templates; this step blocks sync until the user names the components and change kind.
```

### 4. Analyze current implementation

For each affected component, perform targeted codebase exploration. Do **not** read the whole repository exhaustively.

#### 4a. Lightweight project fingerprint (only if not implied by the Canvas)

If the Canvas's Structure section already states concrete file paths and the project's stack is unambiguous from those paths, keep this step brief. Otherwise, detect the stack and tooling from primary dependency/build files, lock files, and obvious tool manifests. List top-level directories and read one obvious relevant configuration file only when useful.

Keep this step fast. Touch only a small number of files. Skip entirely when the Canvas plus the changed files give enough context.

#### 4b. Per-component analysis

Use repository-local tools (`bash` with `rg`, `find`, `ls`, plus `read`) for local code. For questions about third-party APIs, frameworks, or libraries, prefer `code_search`, `web_search`, or the `librarian` skill instead of inventing API details from local grep results.

Read:

- each changed or user-specified source file
- directly related tests when they clarify behavior
- direct dependencies one hop away when required to understand relationships
- relevant build/config files only if needed to understand paths, framework conventions, or validation commands

Extract actual implementation facts:

- file paths and module/package names
- class/type/function/component names
- public methods, signatures, props, routes, commands, event contracts, or schema shapes
- attributes/fields and their types where relevant to the prompt
- annotations/decorators/framework bindings where relevant
- dependencies and call/data flow
- validation rules and exact error messages
- business logic steps and edge/error paths
- tests or assertions that document accepted behavior
- reusable patterns or conventions introduced by the change

Do not infer business intent beyond what the code, tests, prompt, or user context supports.

### 5. Compare implementation against the prompt

For each affected component, locate the corresponding prompt content:

- Entities diagram or Entity Notes
- Approach decisions
- Structure file/component lists and dependency/data-flow notes
- Operations tasks and implementation details
- Norms that describe the pattern used
- Safeguards that constrain behavior, validation, errors, security, compatibility, or tests

Record discrepancies by category:

1. **Structural**: file path, class/module hierarchy, component relationships, dependencies, layering, imports
2. **Behavioral**: business logic, validation, error handling, response/output behavior, edge cases
3. **Naming**: file, class, method, field, route, command, event, or test name changes
4. **Additions**: new fields, methods, modules, components, tests, operations, or extension points
5. **Deletions**: removed fields, methods, modules, operations, or dependencies
6. **Norms**: new or changed coding/testing/observability patterns that should become reusable standards
7. **Safeguards**: changed exact messages, constraints, compatibility requirements, performance/security rules, or test obligations

Classify each discrepancy:

- **Prompt drift**: code is accepted and prompt should be updated to match
- **Potential code defect**: code appears to violate Requirements or Safeguards
- **Requirement/design change**: behavior or architecture changed beyond the current Canvas and needs explicit approval
- **Out of scope**: unrelated code change should not affect this prompt

Number each recorded discrepancy as `D-n` (for example, `D-1`, `D-2`, ...) with its category and classification. Step 6's Sync Plan must reference these IDs.

If a discrepancy is a potential code defect, do not edit the prompt to normalize the defect. Report it and ask whether to fix code through `/spdd-generate` or explicitly update the Canvas.

When stopping for a potential code defect, report it in this form:

```markdown
⚠️ Sync blocked — potential code defect.

- Component: `path/to/file` — [name]
- Canvas section affected: [Requirements/Entities/Approach/Structure/Operations/Norms/Safeguards]
- Why this blocks sync: [reason]
- To resume: either (a) fix the code via `/spdd-generate` against the existing Canvas, then re-run `/spdd-sync`, or (b) explicitly approve a Canvas change that accepts the new behavior. Pi does not auto-chain prompt templates.
```

When stopping for a requirement/design change, report it in this form:

```markdown
⚠️ Sync blocked — requirement/design change beyond current Canvas.

- Change observed: [summary]
- To resume: update the Canvas via `/spdd-prompt-update <prompt-file> <change>` or an explicit user-approved edit to the Canvas file, then re-run `/spdd-sync` for any remaining code-side details. Pi does not auto-chain prompt templates.
```

### 6. Create a prompt sync plan and ask for approval

Before editing the prompt, present a concrete update plan.

Use this format:

```markdown
## Prompt Sync Plan

### Scope
- Prompt: `<prompt-file>`
- Code/components inspected:
  - `path/to/file` — [reason]

### Classification
- Prompt drift to sync: [count]
- Potential code defects: [count]
- Requirement/design changes needing approval: [count]
- Out-of-scope changes ignored: [count]

### Discrepancy Index
- **D-1** [category] [classification] — [observed difference] → [planned action or "no action — see Follow-up"]
- **D-2** ...

### Requirements Section Updates
- [ ] [D-n] No changes recommended unless explicitly approved

### Entities Section Updates
- [ ] [D-n] [Add/update/remove entity, field, method, or relationship]

### Approach Section Updates
- [ ] [D-n] [Update architectural strategy, pattern, or trade-off]

### Structure Section Updates
- [ ] [D-n] [Update file/component/dependency/layer/data-flow description]

### Operations Section Updates
- [ ] [D-n] [Update operation responsibility/signature/logic/validation/error/test details]
- [ ] [D-n] [Add operation for new accepted component]
- [ ] [D-n] [Remove obsolete operation only with explicit approval]

### Norms Section Updates
- [ ] [D-n] [Add/update reusable convention or pattern]

### Safeguards Section Updates
- [ ] [D-n] [Update exact constraints/messages only when implementation and approval support it]

### Approval needed
Please confirm whether to apply this prompt sync plan. Destructive removals, Requirements changes, and Safeguard relaxations require explicit approval.
```

Stop after presenting the plan unless the user's original instruction explicitly authorized applying the plan without another confirmation, using wording such as "sync and apply", "apply the sync plan", or "update the prompt directly". When in doubt, ask.

If the discrepancy set is too large to review in one plan (heuristic: more than ~15–20 items, or spanning unrelated subsystems), propose splitting the sync into multiple smaller approval rounds, such as per subsystem or per REASONS section, and ask the user which slice to apply first.

### 7. Apply approved updates to the prompt file

After approval, update only the structured prompt file. Do not modify source code.

Use `edit` for precise updates to the existing prompt. Use `write` only if the prompt file must be fully rewritten and the user has approved the rewrite.

Apply updates by section:

#### 7a. Requirements

- Requirements are out of scope by default; do not change them unless explicitly approved by the user. Route requirement/design changes through `/spdd-prompt-update <prompt-file> <change>`.
- If Requirements change, preserve AC/DE IDs where possible.
- Do not convert code behavior into a business requirement without explicit approval.

#### 7b. Entities

- Update Mermaid class diagrams or Entity Notes to match actual accepted class/type/data structure.
- Update attributes, method names, and relationship arrows only where they are represented in the existing prompt style.
- Preserve the prompt's abstraction level; do not dump every private implementation detail if the section was intentionally conceptual.

#### 7c. Approach

- Update implementation strategy, architecture pattern, trade-off, or edge-case strategy only when the accepted code demonstrates a meaningful change.
- Keep Approach concise and strategic; put detailed signatures and logic in Operations.

#### 7d. Structure

- Update file paths, create/update/delete intentions, dependencies, data flow, and layering/boundary descriptions.
- Ensure Structure matches actual import/call/dependency relationships for affected components.

#### 7e. Operations

Operations usually require the most detailed sync.

Update as needed:

- Responsibility
- File path or package/module path
- Public API/signature
- Attributes, fields, props, route names, commands, events, or schema contracts
- Logic steps
- Validation and error behavior
- Test cases/assertions
- AC/DE tags when implementation or verification mapping changed
- Completion criteria

Add Operations for newly accepted components. Remove obsolete Operations only with explicit approval.

#### 7f. Norms

- Add or update reusable conventions discovered in accepted implementation.
- Do not add one-off implementation details as Norms.
- Preserve existing numbering/style. Treat `N/A — …` entries as deliberate unless the codebase now makes them applicable.

#### 7g. Safeguards

- Update exact error messages, validation rules, API contracts, compatibility constraints, performance/security constraints, or required tests only when actual accepted code changed and the change is approved.
- Do not relax a Safeguard silently.
- Preserve existing numbering/style. Treat `N/A — …` entries as deliberate unless the accepted implementation changes them.

General editing rules:

- The SPDD prompt file is a specification document, not source code.
- Do **not** include language-specific fenced code blocks such as `java`, `python`, `typescript`, `tsx`, `sql`, `go`, or `rust`.
- Do **not** sync implementation snippets into the Canvas: no class bodies, method bodies, SQL query bodies, decorators/annotations in code form, or full source snippets.
- Allowed fenced code blocks: Mermaid diagrams only, using `mermaid`.
- Keep contracts, signatures, routes, field names, and query descriptions as inline code spans or natural language.
- Preserve the prompt's existing section structure and formatting style.
- Maintain the same level of detail as nearby content.
- Prefer targeted edits over whole-file rewrites.
- Do not simplify, abbreviate, or remove useful detail while syncing.
- Do not change the prompt's unique identifier, title, source analysis line, or metadata unless the user explicitly asks.

### 8. Validate prompt consistency

After updating the prompt, re-read the changed portions or the full file if needed and verify:

1. **Internal consistency**
   - Entities match Structure.
   - Operations reference current component, file, method, field, and route names.
   - Approach explains the current accepted design.
   - Norms are reflected in Operations where applicable.
   - Safeguards appear in relevant Operations/tests.

2. **Prompt-code consistency**
   - Synced prompt details match the inspected implementation.
   - Exact error messages, status codes, validation rules, and public contracts match accepted code.
   - No old class/method/path references remain in affected sections.

3. **Traceability**
   - Build a mapping from each `AC-n` / `DE-n` in Requirements to (a) the Operation(s) that implement it and (b) the test(s) or Safeguard(s) that verify it, using the synced prompt.
   - Every `AC-n` / `DE-n` must still appear on the implementation side and on the verification side.
   - Each affected Structure component has a corresponding Operation or an explicit reason it does not need one.
   - Each affected Operation maps to a component in Structure.
   - If an `AC-n` / `DE-n` no longer maps to anything after sync, classify it as a potential code defect or a requirement/design change instead of silently dropping it.

4. **Specification format and scope control**
   - No language-specific fenced code blocks were introduced, except Mermaid diagrams.
   - Source snippets were not copied into the Canvas.
   - Only approved prompt updates were made.
   - Requirements, destructive removals, and Safeguard relaxations were not changed without explicit approval.
   - The prompt's unique identifier, title, and `_Source analysis: …_` provenance line are unchanged.
   - Source code was not modified.

5. **Diff hygiene (when in a git repo)**
   - Run `git diff --name-only` and confirm only the structured prompt file changed.
   - If any source file appears in the diff, stop and report — `/spdd-sync` must not modify source code.

If validation finds a prompt inconsistency, fix the prompt if it is within the approved plan. If it falls outside the approved plan, report it and ask for approval before making further changes.

### 9. Report sync summary

Reply with:

```markdown
✅ SPDD sync complete for `<prompt-file>`

🧭 Sync scope:
- Code/components inspected:
  - `path/to/file` — [reason]

📝 Prompt sections updated:
- Requirements: changed/unchanged — [summary]
- Entities: changed/unchanged — [summary]
- Approach: changed/unchanged — [summary]
- Structure: changed/unchanged — [summary]
- Operations: changed/unchanged — [summary]
- Norms: [numbered summary, e.g. `1 unchanged, 2 updated, 3 N/A, 4 unchanged`]
- Safeguards: [numbered summary, e.g. `1 unchanged, 2 updated, 3 N/A, 4 unchanged, 5 unchanged`]

🔍 Consistency checks:
- Internal prompt consistency: pass/fail — [summary]
- Prompt-code consistency: pass/fail — [summary]
- AC/DE traceability: pass/fail/not applicable — [summary]
- Scope control: pass/fail — [summary]

🔍 AC/DE coverage after sync:
| ID | Implemented in | Verified by |
|----|----------------|-------------|
| AC-1 | `path/to/file` op N | `path/to/test` or Safeguard n |

⚠️ Follow-up items:
- Potential code defects not synced: [none or list]
- Requirement/design changes needing `/spdd-prompt-update`: [none or list]
- Code generation needed via `/spdd-generate`: [none or list]
- Manual review recommendations: [none or list]
```

If no prompt updates were applied because approval is pending, report the plan and clearly state that no files were modified.

## Sync Patterns

Use these patterns when building the sync plan and applying approved edits.

### Class, function, component, or package renames

- Update names in Entities, Structure, Operations, and relevant Safeguards.
- Update file/package/module paths in Operations and Structure.
- Search affected prompt sections for old names to avoid stale references.
- Preserve old names only when needed for migration/backward-compatibility notes.

### Method signature, route, command, event, or schema changes

- Update Operations public API/signature details.
- Update Entities only if the public model or meaningful relationship changed.
- Update Safeguards if parameters, return values, status codes, exact messages, or constraints changed.
- Update tests/verification descriptions if behavior changed.

### New component added

- Add to Entities if it is a meaningful domain/data/component concept in the prompt's existing abstraction level.
- Add to Structure with responsibility, file path, layer, and dependencies.
- Add an Operation with full implementation-level detail matching the accepted code.
- Update related Operations that depend on the new component.

### Component removed

- Confirm removal is approved before editing.
- Remove or mark obsolete references in Entities, Structure, and Operations according to the prompt's style.
- Update dependent Operations and Safeguards.
- Preserve migration/backward-compatibility notes where relevant.

### Logic or behavior changes

- Update Operations logic steps and tests.
- Update Approach only if the strategy or pattern changed.
- Update Safeguards only when constraints or exact externally visible behavior changed and the change is approved.
- If behavior conflicts with Requirements or Safeguards, stop and classify it as potential code defect or requirement/design change.

### Validation or error-handling changes

- Update Operations validation/error sections.
- Update Safeguards constraints and exact messages only if accepted.
- Ensure tests or manual verification descriptions reflect the new behavior.

### Reusable pattern or convention changes

- Update Norms only when the pattern is reusable beyond a single local implementation detail.
- Keep Norms checkable and consistent with the repository.

## Guardrails

- Do not proceed without reading the entire structured prompt file.
- Do not modify source code.
- Do not update the prompt before presenting a sync plan and receiving approval, unless the user explicitly authorized direct application with wording such as "sync and apply".
- Do not remove prompt content without explicit user approval.
- Do not change Requirements unless explicitly approved.
- Do not relax Safeguards unless explicitly approved and supported by accepted implementation.
- Do not normalize a code defect by changing the prompt to match broken code.
- Do not invent business intent from code alone.
- Do not simplify, abbreviate, or delete detailed specifications while syncing.
- Do not introduce language-specific fenced code blocks except Mermaid diagrams.
- Do not change exact error messages, status codes, validation rules, or public contracts unless they actually changed in accepted code and are approved for sync.
- Do not change the prompt's unique identifier, title, source analysis line, or metadata unless explicitly requested.
- Always preserve the existing formatting style and section structure.
- Always maintain the same level of detail as existing content.
- Always prefer targeted edits over full-file rewrites.
- Always ask for confirmation before destructive changes, Requirements changes, or Safeguard relaxations.
- Always report potential code defects instead of syncing them silently.
- Always validate internal prompt consistency after updates.

## SPDD Workflow Context

This command completes the reverse direction of the local SPDD loop:

```text
Forward flow:        Requirement → /spdd-analysis → /spdd-reasons-canvas → /spdd-generate → Code
Requirement change:  New/changed requirement → /spdd-prompt-update → Updated Canvas → /spdd-generate → Code
Code-side change:    Accepted code change/refactor → /spdd-sync → Updated Canvas
```

Use `/spdd-sync` when:

- code review led to accepted refactoring changes
- implementation discovered a better pattern that the team kept
- a bug fix changed implementation details and must be documented
- performance optimization changed structure or logic
- classes, functions, components, packages, methods, fields, tests, or dependencies were renamed/restructured
- new components were added after the original Canvas
- prompt and code drifted and the code is the accepted source for the current implementation details

Do not use `/spdd-sync` as a substitute for business requirement refinement. If the desired behavior or design should change before code is accepted, update the Canvas through `/spdd-prompt-update` first, then use `/spdd-generate` for implementation.

Note: `/spdd-sync` is sync-only. If the change is a new business requirement or design change, hand off to `/spdd-prompt-update <prompt-file> <change>` before continuing. If the Canvas update implies code changes, hand off to `/spdd-generate <prompt-file>`. Do not inline those workflows inside `/spdd-sync`. Pi does not auto-chain prompt templates.
