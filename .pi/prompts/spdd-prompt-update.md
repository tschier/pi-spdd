---
description: Update an existing SPDD REASONS Canvas prompt
argument-hint: "<@spdd/prompt/file.md or path> <update instructions>"
---

# SPDD Prompt Update

Update an existing SPDD structured prompt using the **REASONS Canvas** framework:

- **R** — Requirements
- **E** — Entities
- **A** — Approach
- **S** — Structure
- **O** — Operations
- **N** — Norms
- **S** — Safeguards

This is the requirement/design-change phase of the SPDD workflow. It updates the saved Canvas when stakeholders, developers, or reviewers intentionally change requirements, architecture, constraints, standards, or specification details.

Source inspiration:

- OpenSPDD `/spdd-prompt-update` command: https://github.com/gszhangwei/open-spdd/blob/main/internal/templates/data/core/spdd-prompt-update.md
- Martin Fowler, "Structured-Prompt-Driven Development (SPDD)": https://martinfowler.com/articles/structured-prompt-driven/

Core SPDD principle:

> When reality diverges, fix the prompt first — then update the code.

## Input

The user's input is:

```text
$ARGUMENTS
```

Input must include:

1. A saved REASONS Canvas prompt file, usually under `spdd/prompt/`
2. Update instructions describing the desired requirement, design, constraint, standard, or specification change

Examples:

```text
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md Add rate limiting: max 100 requests per minute per user
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md Add dependency inversion between service and repository layers
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md Update Safeguards to require exact 409 response for duplicate email
@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md Refine AC-2 so inactive users cannot authenticate
```

Note on Pi references: in Pi, `@file` is usually a UI-level file attachment. If file content is already attached or inlined in the conversation, treat it as already read; do not re-fetch it unless necessary to resolve ambiguity.

## Phase Goal

Modify the existing structured prompt so it accurately expresses the updated intent while preserving the REASONS Canvas structure, provenance, and implementation-ready detail.

The output is the **updated prompt file only**. This prompt does not generate or modify implementation code.

## Phase Boundary

`/spdd-prompt-update` owns only the Requirement/Design Change → Canvas update phase.

It may:

- read and analyze an existing saved REASONS Canvas
- interpret explicit user update instructions
- inspect targeted codebase context when needed to make the Canvas update realistic and project-specific
- update affected REASONS sections in the prompt file
- validate cross-section consistency after the update

It must not:

- modify source code or tests
- sync accepted code-side refactors back into the Canvas; use `/spdd-sync` for Code → Canvas drift
- generate implementation code; use `/spdd-generate` after the Canvas update
- modify files under `spdd/analysis/`; analysis is upstream of the Canvas and is updated only by `/spdd-analysis`
- rename the prompt file or change unrelated prompt sections
- create commits unless the user explicitly asks

If the user's request is really about accepted code that already changed, stop and instruct the user to run `/spdd-sync <prompt-file> <changed files or description>`. If the updated Canvas requires code changes afterward, instruct the user to run `/spdd-generate <prompt-file>`. Pi does not auto-chain prompt templates.

## Steps

### 1. Validate and consolidate input

If no prompt file is provided, stop and ask the user:

> Please provide the path to the SPDD prompt file to update, for example `@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md`.

If no update instructions are provided, stop and ask the user:

> What changes would you like to make to this prompt? For example: new requirements, architectural changes, constraint updates, coding standard changes, or specification corrections.

Do **not** proceed without both a valid prompt file and update instructions.

Resolve the prompt file:

- Treat already attached or inlined `@file` content as read.
- If an argument starts with `@` but no file is attached or inlined, strip the leading `@` and treat the remainder as a plain path.
- If a plain path points to a prompt file, use `read` to read the entire file.
- If a glob is provided for the prompt file, use `bash` to resolve it. Continue only if it resolves to exactly one prompt file; otherwise ask the user to choose.
- If the prompt file cannot be resolved or read, report the problem and ask for a valid path.

Resolve update instructions:

- Treat all remaining text after the prompt path as the update request.
- Treat additional non-prompt `@file` references or paths as supporting requirement/design context. Read them fully unless already attached or inlined.
- If referenced supporting files cannot be read, report the problem and ask for clarification.
- Preserve the user's update intent exactly; do not silently narrow or broaden the requested change.

Before continuing, verify that:

- the entire structured prompt has been read
- every referenced supporting source has been read or surfaced as an error
- the requested update is clear enough to determine affected REASONS sections

### 2. Parse the existing REASONS Canvas

Identify and understand all existing sections:

| Section | Purpose | When to update |
|---------|---------|----------------|
| **Requirements** | Overall goal, scope, DoD, AC/DE IDs | Update when business intent, scope, or acceptance expectations change |
| **Entities** | Domain model, data shapes, relationships | Update when entities, data shapes, relationships, or ownership boundaries change |
| **Approach** | Implementation strategy and trade-offs | Update when architecture, strategy, patterns, or risk responses change |
| **Structure** | Components, files, dependencies, data flow | Update when components, layers, dependencies, or file/module structure change |
| **Operations** | Concrete implementation tasks | Update whenever implementation tasks, signatures, logic, validation, or tests change |
| **Norms** | Engineering standards and project conventions | Update when reusable coding/testing/observability standards change |
| **Safeguards** | Non-negotiable constraints | Update when constraints, exact errors, compatibility, security, performance, or rollout rules change |

If any required REASONS section is missing or malformed, stop and report the issue before editing. Ask whether to repair the Canvas structure as part of this update.

Also identify:

- title and any `_Source analysis: ..._` provenance line
- existing AC/DE IDs and where they are referenced
- existing Mermaid diagram blocks
- existing section formatting, bullet style, numbering style, and level of detail
- any stable Norms/Safeguards numbering and `N/A — ...` entries

### 3. Classify the update request

Determine the update type and affected sections.

| Change type | Usually affected sections |
|-------------|---------------------------|
| New functional requirement | Requirements, Entities, Approach, Structure, Operations, Safeguards, tests; possibly Norms |
| Requirement refinement or AC change | Requirements, Operations, Safeguards, tests; possibly Entities/Approach/Structure |
| Architectural change | Approach, Structure, Operations, Norms; possibly Entities/Safeguards |
| New entity or relationship | Entities, Structure, Operations, tests; possibly Requirements/Approach/Safeguards |
| New constraint or safeguard | Safeguards, Operations, tests; possibly Requirements |
| Coding/testing standard change | Norms, Operations, tests; possibly Safeguards |
| Specification bug fix | Targeted affected section(s), plus consistency updates in related sections |
| Scope reduction or deletion | Requirements, Entities, Structure, Operations, Safeguards, tests — requires explicit confirmation |
| Safeguard relaxation | Safeguards, Operations, tests — requires explicit confirmation |

Classify the requested update as one of:

- **Prompt update**: explicit requirement/design/specification change that should be applied to the Canvas
- **Potential code sync**: accepted implementation already changed and the prompt should reflect code; hand off with `/spdd-sync <prompt-file> <changed files or description>`
- **Potential code generation**: prompt is already correct and code should change; hand off with `/spdd-generate <prompt-file>`
- **Ambiguous**: update instructions are not specific enough; ask for clarification

If ambiguous, stop and ask for clarification using this format:

```markdown
I need clarification before updating the Canvas.

- Prompt: `<prompt-file>`
- Ambiguity: [what is unclear]
- Options:
  1. [interpretation A]
  2. [interpretation B]
- To resume: please choose an option or provide the missing requirement/design detail. Pi does not auto-chain prompt templates.
```

### 4. Read relevant project context when needed

Use targeted codebase exploration only when the update requires project-specific grounding.

Read relevant codebase context if the update involves:

- existing entities, DTOs, models, schemas, routes, components, or APIs
- architecture or layering conventions
- reusable patterns or standards
- integrations, external APIs, queues, jobs, or persistence
- validation, error format, testing style, security, or observability standards

Do **not** read the whole repository exhaustively.

#### 4a. Lightweight project fingerprint (only if not implied by the Canvas)

If the Canvas already states concrete file paths and the project's stack is unambiguous from those paths, keep this step brief. Otherwise, detect the stack and tooling from primary dependency/build files, lock files, and obvious tool manifests. List top-level directories and read one obvious relevant configuration file only when useful.

Keep this step fast. Touch only a small number of files. Skip entirely when the Canvas plus the update instructions provide enough context.

#### 4b. Targeted context reads

Use repository-local tools (`bash` with `rg`, `find`, `ls`, plus `read`) for local code. For questions about third-party APIs, frameworks, or libraries, prefer `code_search`, `web_search`, or the `librarian` skill instead of inventing API details from local grep results.

Read only files directly relevant to the affected sections, such as:

- existing files named in the Canvas Structure/Operations sections
- similar implementations or tests that establish project conventions
- directly related models, validators, routes, services, repositories, components, migrations, or config
- existing SPDD analysis/prompt files only if they are directly relevant to the requested update

Use codebase context to keep the Canvas realistic and project-specific. Do not use code exploration to override explicit business intent without asking the user.

### 5. Build an update plan

Before editing, determine the minimal set of prompt changes required.

Number each proposed change as `U-n` (for example, `U-1`, `U-2`, ...) and map it to the affected REASONS section(s), AC/DE IDs, and source of intent.

Always present a short Update Plan preamble before editing. This preamble is an execution notice, not an approval gate, for ordinary additive or refining updates. Stop for explicit approval **only** when a destructive removal, Requirements scope reduction, Safeguard relaxation, backward-incompatible contract change, contradiction with unchanged requirements, or whole-file rewrite is involved. Otherwise, proceed to Step 6 immediately after the preamble without waiting for a second confirmation.

```markdown
## Prompt Update Plan

### Scope
- Prompt: `<prompt-file>`
- Update request: [summary]
- Classification: `Prompt update` | `Potential code sync` | `Potential code generation` | `Ambiguous` — [one-line reason]
- Supporting context read:
  - `path/to/file` — [reason]

### Proposed updates
- **U-1** — [Requirements/Entities/Approach/Structure/Operations/Norms/Safeguards] — [summary] — Source: [user request/supporting file/code context]
- **U-2** — ...

### Approval needed
- Destructive removals: [none or list]
- Requirements scope reductions: [none or list]
- Safeguard relaxations: [none or list]
- Backward-incompatible contract changes: [none or list]
```

Ask for explicit approval before applying any:

- destructive removal of prompt content, operations, entities, AC/DE IDs, or safeguards
- Requirements scope reduction
- Safeguard relaxation
- backward-incompatible public contract change
- change that contradicts unchanged existing requirements
- whole-file rewrite

If approval is needed, stop and ask:

```markdown
⚠️ Prompt update requires explicit approval.

- Prompt: `<prompt-file>`
- Proposed change(s): [U-n list]
- Why approval is required: [destructive removal / scope reduction / Safeguard relaxation / incompatible change / contradiction]
- To resume: confirm which proposed changes to apply, revise the update request, or cancel. Pi does not auto-chain prompt templates.
```

For ordinary additive or refining updates that are clearly requested by the user and not destructive, proceed without a second confirmation.

If the update set is too large to review or apply safely in one pass (heuristic: more than ~15–20 proposed changes, or spanning unrelated subsystems), propose splitting the update into multiple smaller rounds and ask the user which slice to apply first.

### 6. Apply updates to affected sections only

Update the existing prompt file in place. Preserve the original filename.

Use `edit` for targeted updates to the existing prompt. Use `write` only if a whole-file rewrite is explicitly approved.

General update rules:

- **Minimal change**: only modify what is necessary to satisfy the user's update intent.
- **Preserve intent**: do not change the original design intent unless the user explicitly requested it.
- **Backward compatibility**: consider impact on any existing implementation; if the change is incompatible, route through the approval gate in Step 5.
- Modify only sections affected by the update request and required consistency changes.
- Preserve unchanged content verbatim whenever possible.
- Preserve the existing section structure, heading levels, formatting style, bullet style, and level of detail.
- Preserve the prompt title, `_Source analysis: ..._` provenance line, and any unique identifier or metadata unless explicitly asked.
- Do not rename the file.
- Do not leave placeholders, TODOs, unresolved template variables, or vague generic statements.
- Do not simplify, abbreviate, or delete detailed specifications while updating.
- Keep changes traceable to the update request and `U-n` plan items.
- Every applied edit must trace back to a `U-n` plan item; if a required edit has no plan ID, add it to the plan before applying.

#### 6a. Requirements

Update Requirements when business goal, scope, non-goals, DoD, or AC/DE expectations change.

- Preserve existing AC/DE IDs where possible.
- Add new AC/DE IDs using the existing numbering style.
- Do not renumber existing AC/DE IDs unless explicitly required; prefer appending or marking superseded details in the existing style.
- If an AC/DE changes, update corresponding Operations, Safeguards, and test expectations.
- Scope reductions and deleted AC/DE IDs require explicit approval.

#### 6b. Entities

Update Entities when domain/data concepts, input/output shapes, relationships, ownership, or lifecycle boundaries change.

- Update Mermaid diagrams only when the prompt already uses Mermaid and the relationship/model change is meaningful at that level; see Step 7 for allowed fenced code block rules.
- Update Entity Notes consistently with diagrams.
- Prefer minimal model changes that satisfy the request.
- Avoid adding wrapper entities, abstractions, or persistence concepts unless required by the update.

#### 6c. Approach

Update Approach when strategy, architecture, pattern, data/control flow, edge-case handling, compatibility stance, or trade-offs change.

- Keep this section strategic and concise.
- Put detailed signatures, step-by-step logic, and tests in Operations.
- Record important alternatives or trade-offs only when they affect implementation choices.

#### 6d. Structure

Update Structure when files, components, dependencies, layers, public boundaries, data flow, migrations, jobs, events, integrations, or UI components change.

- Use actual project paths and naming conventions when known.
- Keep dependency direction explicit.
- Ensure every new or changed component has a corresponding Operation or a clear reason it does not need one.
- Avoid unnecessary refactors or new layers not required by the update.

#### 6e. Operations

Operations usually require detailed updates whenever the Canvas changes.

Update as needed:

- operation ordering and dependency sequence
- responsibility
- file path or package/module path
- create/update/delete intent
- public API/signature, route, command, event, props, schema, or migration contract
- fields/attributes and meaningful type details
- logic steps
- validation and exact error behavior
- logging/observability/security behavior
- tests and assertions
- AC/DE tags
- completion criteria

Do not re-plan unrelated Operations. Keep existing Operation order unless the update requires a dependency-order change. If order changes, ensure dependencies remain valid.

#### 6f. Norms

Update Norms when reusable engineering standards change.

- Preserve stable numbering and categories.
- Treat `N/A — ...` entries as deliberate unless the update makes that category applicable.
- Add standards only when reusable and checkable.
- Prefer repository-specific conventions over generic best practices.
- If a Norm changes, update Operations that must follow it.

#### 6g. Safeguards

Update Safeguards when constraints, exact messages, validation rules, API contracts, security/privacy requirements, performance limits, compatibility, rollout, or testing obligations change.

- Preserve stable numbering and categories.
- Treat `N/A — ...` entries as deliberate unless the update makes that category applicable.
- Do not relax Safeguards without explicit approval.
- Preserve exact error messages and public contracts unless the update explicitly changes them.
- If a Safeguard changes, update corresponding Operations and tests.

### 7. No language-specific code blocks in the updated prompt

The SPDD prompt file is a **specification document**, not source code. It describes what to implement; `/spdd-generate` produces actual source code later.

In the updated prompt file:

- Do **not** include language-specific fenced code blocks such as `java`, `python`, `typescript`, `tsx`, `sql`, `go`, or `rust`.
- Do **not** include implementation code: no class bodies, method bodies, SQL query bodies, decorators/annotations in code form, or full source snippets.
- Use natural language and inline code spans for contracts and signatures.
- Allowed fenced code blocks: Mermaid diagrams only, using `mermaid`.

Use this style:

- ✅ Method signature: "Method `findById(String id)` returns `Optional<Customer>` and returns empty when no customer exists."
- ✅ Query logic: "Query active subscriptions where `customerId` matches and the usage date falls within the effective range, ordered by `createdAt` descending."
- ✅ Interface contract: "Repository interface defines `save(Bill)` and `findByCustomerId(String)` operations."
- ❌ Do not include a fenced source-code block containing a repository class, SQL statement, or method body.

### 8. Validate cross-section consistency

After updating, re-read the changed portions or the full prompt if needed and verify:

1. **REASONS structure**
   - All seven sections still exist.
   - Heading levels and section names remain consistent with the existing prompt.
   - Prompt title, filename, unique identifier/metadata, and `_Source analysis: ..._` provenance line are preserved unless explicitly changed.
   - If a provenance line like `_Source analysis: ..._` references `spdd/analysis/...`, optionally verify the referenced file still exists; if it does not, add a Follow-up note rather than failing the update.

2. **Internal consistency**
   - Requirements are reflected in Approach, Structure, Operations, Safeguards, and tests.
   - Entities mentioned in Operations and Structure exist in Entities or are intentionally external/transient.
   - Dependencies and data flow in Structure match Operations.
   - Norms are applied in Operations where relevant.
   - Safeguards are enforceable through Operations and tests.
   - No stale names, paths, methods, fields, routes, or AC/DE references remain in affected sections.

3. **AC/DE traceability**
   - Build a mapping from each `AC-n` / `DE-n` in Requirements to (a) the Operation(s) that implement it and (b) the test(s), Safeguard(s), or manual check(s) that verify it.
   - Every new or changed `AC-n` / `DE-n` must appear on the implementation side and verification side.
   - If any existing `AC-n` / `DE-n` becomes orphaned, fix the related sections or ask for clarification.

4. **Specification quality**
   - No placeholders, TODOs, unresolved template variables, or vague generic content remain.
   - The update is specific enough for `/spdd-generate` to implement without inventing details.
   - No language-specific fenced code blocks were introduced, except Mermaid diagrams.
   - If any Mermaid block was added or modified, confirm the block is syntactically valid: balanced braces, valid relationship arrows, and recognized diagram type.

5. **Scope control**
   - Only affected sections and required consistency updates changed.
   - Destructive removals, scope reductions, Safeguard relaxations, and incompatible contract changes were not made without explicit approval.
   - Source code was not modified.

6. **Diff hygiene (when in a git repo)**
   - Run `git diff --name-only` and confirm only the structured prompt file changed.
   - If any source file appears in the diff, stop and report — `/spdd-prompt-update` must not modify source code.
   - Confirm no file under `spdd/analysis/` appears in the diff. If one does, stop and report — `/spdd-prompt-update` must not modify analysis artifacts.

If validation finds an inconsistency caused by the update, fix the prompt if it is within scope. If fixing it requires a new decision or approval, stop and ask.

### 9. Report update summary

Reply with:

```markdown
✅ SPDD prompt updated: `<prompt-file>`

📋 Update request:
- [brief summary]
- Classification: [Prompt update / Potential code sync / Potential code generation / Ambiguous] — [one-line reason]

📌 Applied updates:
| ID | REASONS section(s) | Source of intent | Status |
|----|--------------------|------------------|--------|
| U-1 | Requirements, Operations | user request | applied |
| U-2 | Safeguards | supporting file `path/...` | deferred — pending approval |

📝 Changes made:
- Requirements: changed/unchanged — [summary]
- Entities: changed/unchanged — [summary]
- Approach: changed/unchanged — [summary]
- Structure: changed/unchanged — [summary]
- Operations: changed/unchanged — [summary]
- Norms: [numbered summary if numbered, otherwise changed/unchanged — summary]
- Safeguards: [numbered summary if numbered, otherwise changed/unchanged — summary]

🔍 AC/DE coverage after update:
| ID | Implemented in | Verified by |
|----|----------------|-------------|
| AC-1 | `path/to/file` op N | `path/to/test`, Safeguard n, or manual check |

🔍 Validation:
- REASONS structure: pass/fail — [summary]
- Internal consistency: pass/fail — [summary]
- Specification quality: pass/fail — [summary]
- Scope control: pass/fail — [summary]

⚠️ Follow-up items:
- Clarifications needed: [none or list]
- Manual review recommendations: [none or list]
- Code generation needed via `/spdd-generate`: [none or list]
- Code sync needed via `/spdd-sync`: [none or list]
```

If no prompt updates were applied because clarification or approval is pending, report the proposed plan and clearly state that no files were modified.

After reporting, include this user-driven handoff when code changes are needed:

```markdown
🔗 Next step: Regenerate affected code explicitly:
   /spdd-generate <prompt-file>
```

Do not invoke `/spdd-generate` automatically. Pi does not auto-chain prompt templates.

## Update Patterns

Use these patterns when deciding which sections to update.

### Adding a new functional requirement

- Update Requirements with a concise requirement and AC/DE IDs.
- Update Entities if new domain/data concepts or shapes are needed.
- Update Approach with strategy and trade-offs.
- Update Structure with affected files/components and dependencies.
- Add or modify Operations for implementation and tests.
- Add Safeguards for constraints, exact errors, security, compatibility, and verification.
- Update Norms only if a reusable standard changes.

### Refining an acceptance criterion or business rule

- Preserve AC/DE IDs where possible.
- Update the requirement text and all related Operations, Safeguards, and tests.
- If the old behavior must remain for compatibility, state the compatibility rule explicitly.

### Adding architectural principles

- Update Approach with the principle and rationale.
- Update Structure with dependency direction, boundaries, or component split.
- Update Operations affected by the architecture change.
- Update Norms if the principle becomes a reusable standard.
- Add Safeguards only for non-negotiable architectural constraints.

### Adding a new entity or relationship

- Update Entities diagram/notes.
- Update Structure with ownership, dependencies, and data flow.
- Add Operations for creating/updating affected files and tests.
- Update Safeguards for data integrity and compatibility constraints.

### Adding or changing constraints

- Update Safeguards first.
- Update Operations validation, errors, security, performance, rollout, or tests so Safeguards are enforceable.
- Update Requirements only if the constraint changes externally visible acceptance expectations.

### Updating coding or testing standards

- Update Norms with reusable, checkable standards.
- Update Operations that must follow the new standard.
- Do not add one-off implementation details as Norms.

### Specification bug fix

- Update only the incorrect section(s) and required consistency references.
- Preserve original intent unless the user explicitly changes it.
- Validate that no old contradictory text remains.

### Scope reduction or deletion

- Ask for explicit approval before removing entities, operations, AC/DE IDs, safeguards, or requirements.
- Preserve migration, compatibility, or deprecation notes where relevant.
- Update all references to removed concepts.

## Guardrails

- Do not proceed without both a valid prompt file and clear update instructions.
- Do not modify source code or tests.
- Do not use `/spdd-prompt-update` to sync accepted code-side refactors; use `/spdd-sync` for Code → Canvas drift.
- Do not generate implementation code.
- Do not rename the prompt file.
- Do not rewrite the entire prompt unless explicitly approved.
- Do not change unaffected sections except for required consistency updates.
- Do not remove prompt content, reduce scope, delete AC/DE IDs, or relax Safeguards without explicit approval.
- Do not change exact error messages, status codes, validation rules, security constraints, compatibility requirements, or public contracts unless explicitly requested.
- Do not leave placeholders, TODOs, unresolved template variables, or vague generic content.
- Do not introduce language-specific fenced code blocks except Mermaid diagrams.
- Do not invent business intent from codebase exploration.
- Preserve the REASONS Canvas structure and all seven sections.
- Preserve prompt title, filename, unique identifier/metadata, and `_Source analysis: ..._` provenance line unless explicitly requested.
- Preserve existing formatting style, heading levels, numbering, bullet style, and level of detail.
- Preserve stable AC/DE IDs where possible; avoid renumbering unless explicitly required.
- Preserve stable Norms/Safeguards numbering and treat `N/A — ...` entries as deliberate unless the update makes them applicable.
- Prefer targeted `edit` calls over full-file rewrites.
- Validate cross-section consistency after updates.
- Always report what changed and what remained unchanged.
- Do not create commits unless explicitly asked.

## SPDD Workflow Context

This command supports the requirement/design-change arm of the local SPDD loop:

```text
Create Canvas:        Requirement → /spdd-analysis → /spdd-reasons-canvas → Canvas
Requirement change:   New/changed requirement → /spdd-prompt-update → Updated Canvas → /spdd-generate → Code
Code-side change:     Accepted code change/refactor → /spdd-sync → Updated Canvas
Forward generation:   Updated Canvas → /spdd-generate → Code
```

Use `/spdd-prompt-update` when:

- stakeholders add or refine requirements
- acceptance criteria or derived expectations change
- architectural decisions are refined before code generation
- constraints, exact error behavior, compatibility rules, or rollout rules change
- coding/testing standards need to be added to the Canvas
- a specification bug is discovered and the prompt should be fixed before code changes

Do not use `/spdd-prompt-update` as a substitute for `/spdd-sync` when the code already changed and the prompt needs to reflect accepted implementation details. Do not use it as a substitute for `/spdd-generate` when the prompt is already correct and source code needs to change.

Note: `/spdd-prompt-update` is prompt-update-only. If accepted code drift needs to be synchronized, hand off to `/spdd-sync <prompt-file> <changed files or description>`. If implementation changes are needed after the Canvas update, hand off to `/spdd-generate <prompt-file>`. Do not inline those workflows inside `/spdd-prompt-update`. Pi does not auto-chain prompt templates.
