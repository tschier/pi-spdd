---
description: Create a REASONS Canvas prompt from SPDD context
argument-hint: "<@spdd/analysis/file.md, requirement text, path, glob, or folder>"
---

# SPDD REASONS Canvas

Generate a structured implementation prompt using the **REASONS Canvas** framework:

- **R** — Requirements
- **E** — Entities
- **A** — Approach
- **S** — Structure
- **O** — Operations
- **N** — Norms
- **S** — Safeguards

This is the tactical SPDD phase. Transform an enriched SPDD analysis or business requirement into a complete implementation prompt. This prompt itself does **not** implement code; implementation belongs to a later workflow after this artifact has been saved and the user has explicitly confirmed how to proceed.

Source inspiration:

- OpenSPDD `/spdd-reasons-canvas` command: https://github.com/gszhangwei/open-spdd/blob/v0.4.9/internal/templates/data/core/spdd-reasons-canvas.md
- Martin Fowler, "Structured-Prompt-Driven Development (SPDD)": https://martinfowler.com/articles/structured-prompt-driven/

## Input

The user's input is:

```text
$ARGUMENTS
```

Input may contain:

1. A saved SPDD analysis file, usually from `spdd/analysis/`
2. Direct text describing the requirement
3. Pi `@file` references to requirement or analysis documents
4. Plain file paths, folder paths, or glob patterns
5. A combination of text and references

Note on Pi references: in Pi, `@file` is usually a UI-level file attachment. If file content is already attached or inlined in the conversation, treat it as already read; do not re-fetch it unless necessary to resolve ambiguity.

## Phase Goal

Produce a fully populated, operationally usable REASONS Canvas prompt and save it under:

```text
spdd/prompt/
```

The saved prompt should be suitable for a later implementation phase. It must contain concrete enough guidance for implementation, while still requiring explicit user confirmation before code changes begin.

## Steps

### 1. Validate and consolidate input context

If no input is provided, stop and ask the user:

Please provide the SPDD analysis file or business requirement description. You can use text, `@file` references, plain paths, folders, or globs.

Do **not** proceed without input context.

If the consolidated input includes file paths, folder paths, globs, or `@`-style references:

- Treat any Pi-attached or already-inlined file content as already read; do not re-fetch it unnecessarily.
- If an `@file` reference is not attached or inlined, strip the leading `@` and treat the remainder as a plain path.
- For any path or glob that is not already attached or inlined, enumerate with `bash`/`find`/`rg --files` and read the relevant files.
- For folder inputs, list the folder and read all relevant requirement, analysis, design, or specification files inside.
- If a reference cannot be resolved or read, report the problem and ask the user for an alternative.
- Combine all referenced source contents with any direct text input.
- Preserve the full original meaning and information.
- Do not summarize or truncate the input context.

If the consolidated input contains an SPDD Analysis document, recognizable by a `# SPDD Analysis:` header and the sections `## Domain Concept Identification`, `## Strategic Approach`, and `## Risk and Gap Analysis`, reuse its concepts, decisions, risks, and acceptance-criteria coverage directly as the basis for the REASONS Canvas. Do not re-derive strategic analysis from scratch unless the source analysis is incomplete or contradicted by current codebase findings. Track the source analysis filename for provenance in the saved prompt.

Before continuing, verify that every referenced source ended up in the consolidated context, either via Pi attachment/inlining or explicit reading. Never proceed with partial referenced content.

Context integrity checklist — confirm in reasoning before Step 2:

- [ ] Every `@file`, path, folder, or glob in the input either matched files that were read, was already attached/inlined, or surfaced an error to the user.
- [ ] No glob or folder input silently expanded to zero relevant files.
- [ ] No referenced file was summarized in lieu of being read or attached in full.
- [ ] The consolidated context preserves the original requirement intent and any source analysis content.

### 2. Perform implementation-oriented codebase exploration

Use targeted exploration. Do **not** read the entire codebase exhaustively.

For repository-local searches, use `bash` with tools such as `rg`, `find`, and `ls` (or Pi's built-in `grep`/`find`/`ls` tools if available). Use external library/API search tools only for questions about third-party APIs, not for code in this repository. For third-party API, framework, or library questions, prefer `code_search`, `web_search`, or the `librarian` skill instead of inventing API details from local grep results.

#### 2a. Lightweight project fingerprint

Always begin with a lightweight project fingerprint:

- Detect the stack from primary dependency/build files, lock files, and obvious tool manifests.
- List top-level directories to understand project layout.
- Read obvious relevant configuration files only when it materially helps identify framework conventions, runtime settings, or validation commands.

Keep this step fast. Touch only a small number of files; do not enumerate every possible ecosystem manifest unless needed to disambiguate the project.

#### 2b. Extract implementation search concepts

From the consolidated context, extract:

- **Domain nouns**: likely entities or concepts, e.g. customer, bill, usage, pricing plan, quota
- **Action verbs**: likely operations, e.g. submit usage, calculate bill, export report
- **API surfaces**: endpoints, events, commands, queues, routes, screens, jobs, or integrations explicitly mentioned
- **Technical hints**: mentioned technologies, patterns, modules, or domain-specific terms
- **Acceptance criteria**: explicit ACs, testable requirement statements, or expected behaviors

Use these concepts as the search scope for codebase exploration.

#### 2c. Targeted schema/model/type exploration

Within the concept scope:

- Search database migrations, schema files, ORM models, entity classes, type definitions, validators, serializers, and DTO-like structures using repository-local search (`rg`, `find`, `ls`, or equivalent).
- Read only matched files and directly relevant definitions.
- Follow foreign-key, ownership, import, or type relationships one hop outward from matched concepts.
- If no matching schema/model/type code exists, note that explicitly.

#### 2d. Targeted implementation-pattern exploration

Within the concept scope:

- Search filenames, class names, functions, services, controllers, routes, handlers, tests, jobs, modules, and UI components using repository-local search (`rg`, `find`, `ls`, or equivalent).
- Read matched files to understand existing business logic, validation, error handling, layering, dependency injection, API style, test style, and naming conventions.
- Follow direct dependencies one hop, but do not recursively expand indefinitely.
- Observe actual project conventions before proposing operations, file paths, names, signatures, or tests.
- If the area appears greenfield, state that and rely only on framework/project conventions discovered from files.

#### 2e. Relevant existing SPDD context

If present, inspect:

```text
spdd/analysis/
spdd/prompt/
```

Read only files whose names or contents appear relevant to the extracted concepts.

#### 2f. Controlled expansion

If exploration reveals one clearly essential concept that was missing from the initial extraction, add it to the concept list and do one additional targeted search round.

Do not expand beyond one extra hop. At most one additional concept may be added, triggering one additional search round: one to three `rg`/`find` commands and the file reads they directly motivate, capped at roughly five additional file reads. Beyond that, note the exploration boundary explicitly in the saved prompt if important context may remain outside the explored scope.

### 3. Apply the REASONS Canvas framework

Generate fully populated content for each of the seven stages below. Adapt all examples and terminology to the detected stack and actual project conventions. Do not hardcode Java, Spring, REST, SQL, or any other stack unless the codebase or requirement indicates it.

---

#### R — Requirements

**Objective**: Extract the core problem essence, value, boundaries, and acceptance expectations.

**Output format in the saved prompt**:

```markdown
## Requirements
- [Concise requirement statement]
- [Business value or user outcome]
- [Boundary or non-goal]
- [Acceptance expectation or measurable behavior]
```

**Construction guidance**:

- Abstract the fundamental problem to solve and the value to create.
- Clarify scope, non-goals, and externally imposed constraints.
- Prefer concise verb phrases such as "Implement...", "Create...", "Validate...", "Expose...", "Prevent...".
- Connect each requirement to the user or business outcome it enables; avoid feature stacking.
- Preserve explicit acceptance criteria where present.
- Assign stable IDs to explicit acceptance criteria (`AC-1`, `AC-2`, ...) and derived expectations (`DE-1`, `DE-2`, ...) so Operations, Tests, and Safeguards can reference them.
- If no explicit acceptance criteria exist, derive testable expectations from requirement statements and label them as derived expectations.

**Quality standards**:

- Core requirement can be summarized in one sentence.
- Business value is clear.
- Scope boundaries are explicit.
- Requirements are testable or can be mapped to safeguards/tests.

---

#### E — Entities

**Objective**: Define the business entities, data shapes, and relationships needed for implementation.

**Output format in the saved prompt**:

Include a Mermaid `classDiagram` when there are two or more entities or non-trivial relationships. For trivial single-entity changes or pure workflow changes, omit the diagram and document the entity, transient shapes, or absence of entities in prose under `### Entity Notes`.

````markdown
## Entities

```mermaid
classDiagram
direction TB

class [CoreEntity] {
  +[type] [attributeName]
  +[methodName]([params]) [returnType]
}

class [RelatedEntity] {
  +[type] [attributeName]
}

class [InputShape] {
  +[type] [fieldName]
}

class [OutputShape] {
  +[type] [fieldName]
}

[CoreEntity] "[cardinality]" -- "[cardinality]" [RelatedEntity] : [relationship]
[InputShape] --> [CoreEntity] : maps to
[CoreEntity] --> [OutputShape] : maps to
```

##### Entity Notes
- **[EntityName]**: [purpose, lifecycle, ownership, important invariants]
````

**Construction guidance**:

- Identify core business entities, supporting entities, request/input shapes, response/output shapes, persistence records, events, or UI view models as appropriate for the project.
- Use names, attributes, and types that match actual project conventions where discoverable.
- Model important relationships, ownership boundaries, cardinalities, and data flow.
- Include key methods only when they are meaningful at prompt/design level and align with the codebase style.
- Include existing entities and new entities distinctly when useful.

**Conservative constraints (critical)**:

- Prefer extending existing structures over creating new abstractions.
- Do not introduce wrapper entities if existing simple types or structures are sufficient.
- Do not propose schema/entity refactors unless the requirement cannot be met through existing structures.
- Preserve backward compatibility unless the requirement explicitly allows breaking changes.
- If the change requires no persistent entity, state that explicitly and model only transient input/output or workflow concepts.

**Quality standards**:

- Entity relationships are clear and justified.
- Models are as simple as possible while satisfying the requirement.
- Proposed types align with the detected stack and project conventions.
- Over-abstraction is avoided.

---

#### A — Approach

**Objective**: Provide concrete solution strategies and key design decisions.

**Output format in the saved prompt**:

```markdown
## Approach

### 1. [Solution Area]
- [High-level strategy]
- [Architecture or integration pattern]
- [Rationale and trade-off]

### 2. [Technical Implementation]
- [Framework/library/project convention to use]
- [Validation, persistence, API, UI, job, or integration approach]
- [Error handling and observability strategy]

### 3. [Business Logic]
- [Core business rules]
- [Workflow/process design]
- [Edge-case strategy]
```

**Construction guidance**:

- Organize by solution categories relevant to the requirement, such as API, UI, data processing, state management, persistence, validation, jobs, events, integrations, exceptions, testing, or observability.
- Explain key architecture choices and why they fit the existing codebase.
- Identify important trade-offs and alternatives rejected.
- Include risk responses where implementation direction matters.
- If the change touches existing public contracts, persisted schemas, queues/events, stored data, URLs, CLI commands, or other integration boundaries, explicitly state the backward-compatibility stance: preserve, additive-only, or breaking with migration. Also state where Operations and Safeguards enforce that stance.
- Use project-specific patterns discovered during exploration.

**Quality standards**:

- Solution direction is operationally usable.
- Key technical decisions are explicit.
- The approach is coherent with existing architecture.
- Risks and edge cases are accounted for.

---

#### S — Structure

**Objective**: Define the concrete component, module, file, and dependency structure for implementation.

**Output format in the saved prompt**:

```markdown
## Structure

### Files and Components
1. `[path/to/file]` — [create/update/delete] — [responsibility]
2. `[path/to/file]` — [create/update/delete] — [responsibility]

### Dependencies and Data Flow
1. [ComponentA] calls/uses [ComponentB] to [purpose]
2. [Input] flows through [validation/service/domain/persistence/output]

### Layering / Boundaries
1. [Layer or module]: [responsibility]
2. [Layer or module]: [responsibility]
```

**Construction guidance**:

- Use actual project paths and naming conventions whenever possible.
- Include create/update/delete intentions for files or components.
- Define dependency direction and call flow.
- Respect layering boundaries observed in the codebase.
- Include extension points only when justified.
- If inheritance/interfaces are idiomatic in the project, describe them; otherwise do not force them.

**Quality standards**:

- File/component plan is concrete.
- Dependencies are clear and avoid circular or inappropriate coupling.
- Structure supports testing and future extension without unnecessary abstraction.

---

#### O — Operations

**Objective**: Transform the approach and structure into specific executable implementation tasks.

**Output format in the saved prompt**:

```markdown
## Operations

### 1. Create/Update [ComponentType] — `[path/to/file]` `(AC: AC-1; DE: DE-2)`
- **Responsibility**: [clear responsibility]
- **Changes**:
  - [specific change]
  - [specific change]
- **Public API / Signature**:
  - `[functionOrMethodName]([parameters]): [returnType]` — [contract]
- **Logic**:
  1. [step]
  2. [step]
  3. [edge/error path]
- **Validation / Errors**:
  - [validation rule]
  - [error behavior]
- **Completion criteria**:
  - [verifiable criterion]

### 2. Add/Update Tests — `[path/to/test-file]` `(AC: AC-1; DE: DE-2)`
- **Cases**:
  - [happy path]
  - [edge case]
  - [failure case]
- **Assertions**:
  - [expected result]
```

**Construction guidance**:

- Base operations strictly on Requirements, Entities, Approach, and Structure.
- Group tasks by module/component and order them by dependency.
- Include exact file paths when reasonably inferable from the codebase.
- Include method/function signatures, command names, route names, schema changes, migration names, component props, or event contracts when appropriate to the detected stack.
- Include validation, error handling, logging/observability, migration, data compatibility, and test tasks when relevant.
- Tag Operations and test headings with the standard syntax `(AC: AC-n; DE: DE-n)` when they implement or verify acceptance criteria and derived expectations. Use only the relevant part when one side is absent, e.g. `(AC: AC-1)` or `(DE: DE-2)`.
- Each task must have a clear responsibility and completion criteria.
- If a detail cannot be inferred safely, state the decision point explicitly instead of inventing a false fact.

**Quality standards**:

- Tasks can be executed directly by an implementation agent or developer.
- Implementation order is logical.
- Acceptance expectations are covered by tasks and tests.
- Details are specific but not fabricated.

---

#### N — Norms

**Objective**: Define coding standards, implementation conventions, and reusable patterns that must be followed.

**Output format in the saved prompt**:

```markdown
## Norms
1. **Project conventions**: [naming, formatting, module, dependency, or style conventions]
2. **Validation**: [validation patterns and error reporting]
3. **Error handling**: [exception/result/error response strategy]
4. **Testing**: [test framework, style, fixtures, coverage expectations]
5. **Logging / observability**: [logging, metrics, tracing, audit requirements]
6. **Documentation**: [comments, API docs, README, generated docs]
7. **Security / privacy**: [handling of secrets, PII, auth, authorization]
```

**Construction guidance**:

- Derive standards from existing code whenever possible.
- Keep the main Norms numbering stable. If a category is not applicable, mark it `N/A — [reason]` rather than deleting or renumbering it.
- Include only norms relevant to the requirement and detected stack within those stable categories.
- Make standards checkable.
- Prefer consistency with the repository over generic best practices.
- Include formatting/lint/test commands if discovered.

**Quality standards**:

- Norms are concrete, enforceable, and project-specific.
- Patterns are reusable without being over-prescriptive.
- Quality and consistency expectations are clear.

---

#### S — Safeguards

**Objective**: Define boundary conditions, constraints, risks, and verification criteria that protect implementation quality.

**Output format in the saved prompt**:

```markdown
## Safeguards
1. **Functional constraints** `(AC: AC-n; DE: DE-n)`: [specific functional boundaries]
2. **Data constraints** `(AC: AC-n; DE: DE-n)`: [data validity, migration, compatibility, retention]
3. **Security constraints** `(AC: AC-n; DE: DE-n)`: [auth, authorization, privacy, injection, exposure]
4. **Performance constraints** `(AC: AC-n; DE: DE-n)`: [latency, complexity, throughput, resource limits]
5. **Integration constraints** `(AC: AC-n; DE: DE-n)`: [API contracts, backward compatibility, external dependencies]
6. **Error-handling constraints** `(AC: AC-n; DE: DE-n)`: [classified errors, no sensitive leakage, retry/idempotency]
7. **Testing constraints** `(AC: AC-n; DE: DE-n)`: [required coverage, regression scenarios, manual checks]
8. **Rollout constraints** `(AC: AC-n; DE: DE-n)`: [migration, feature flag, deployment, rollback]
```

**Construction guidance**:

- Define what can and cannot be done.
- Make constraints verifiable.
- Cover relevant functional, data, security, performance, integration, compatibility, and rollout concerns.
- Quantify constraints where possible.
- Keep the main Safeguards numbering stable. If a category is not applicable, mark it `N/A — [reason]` rather than deleting or renumbering it.
- Tag Safeguards with the standard syntax `(AC: AC-n; DE: DE-n)` when they enforce acceptance criteria or derived expectations. Use only the relevant part when one side is absent.
- Tie safeguards back to acceptance criteria, risks, and implementation tasks.

**Quality standards**:

- Constraints are clear and testable.
- Safeguards cover the requirement's main risk areas.
- No sensitive or unsafe behavior is introduced.

### 4. Construct the final REASONS Canvas prompt

Create one Markdown document with this structure:

````markdown
# [Derived Requirement Title]

_Source analysis: `spdd/analysis/<filename>.md`_ (include only when the input was a saved SPDD Analysis document)

## Requirements
[fully populated content]

## Entities
[fully populated content; include a Mermaid diagram only when there are two or more entities or non-trivial relationships]

## Approach
[fully populated content]

## Structure
[fully populated content]

## Operations
[fully populated implementation tasks]

## Norms
[fully populated standards]

## Safeguards
[fully populated constraints]
````

The saved prompt must include only the generated REASONS Canvas content. It is a specification document, not source code.

Do **not** include:

- The original business context or SPDD analysis verbatim unless a small excerpt or source-analysis provenance line is necessary for clarity
- Framework metadata such as Objective, Construction Guidance, or Quality Standards
- Generation timestamp or framework name as standalone metadata
- Empty sections, placeholders, TODOs, or unresolved template variables
- Language-specific fenced code blocks such as `java`, `python`, `typescript`, `tsx`, `sql`, `go`, or `rust`
- Implementation code: no class bodies, method bodies, SQL query bodies, decorators/annotations in code form, or full source snippets

Allowed fenced code blocks: Mermaid diagrams only, using `mermaid`. Keep contracts, signatures, routes, field names, and query descriptions as inline code spans or natural language.

### 5. Save the fully populated structured prompt

Create `spdd/prompt/` if it does not exist.

Derive a filename using:

```text
{TICKET}-{TIMESTAMP}-[{ACTION}]-{scope}-{description}.md
```

Rules:

- **TICKET**: Extract a ticket/Jira-like identifier from the input context if present. Otherwise use `SPDD-XXX`.
- **TIMESTAMP**: Use current UTC time as `YYYYMMDDHHmm`. Use `bash` to obtain it, for example: `date -u +%Y%m%d%H%M`.
- **ACTION**: Infer from the context using one of `[Feat]`, `[Fix]`, `[Refactor]`, `[Test]`, or `[Docs]`.
- **scope**: Infer from the context, such as `api`, `ui`, `service`, `repo`, `db`, `job`, `event`, `integration`, `config`, `docs`, or `util`. Omit if no clear scope exists.
- **description**: Derive from the context, kebab-case, fewer than 10 words.
- **collision handling**: Never overwrite an existing `spdd/prompt/` file from this canvas-generation phase. If the exact target file or another prompt for the same ticket and requirement already exists, append `-v2`, `-v3`, and so on to create a new versioned artifact. Use `/spdd-prompt-update` for intentional in-place edits to an existing Canvas.

Examples:

```text
SPDD-XXX-202603131530-[Feat]-api-user-registration.md
ABC-169-202603131530-[Fix]-service-payment-validation.md
JIRA-42-202603131530-[Refactor]-db-usage-rollups.md
```

Write the full document to:

```text
spdd/prompt/<filename>.md
```

This is the only file modification allowed by this prompt. Do not modify source code or existing project files.

### 6. Report completion

After saving, reply with:

```markdown
✅ REASONS Canvas prompt generated and saved to `spdd/prompt/<filename>.md`

📋 Generated sections:
- Requirements: [1-line summary]
- Entities: [entity count or "no persistent entities"]
- Approach: [main approach summary]
- Structure: [component/file count and architecture pattern]
- Operations: [task count] implementation tasks
- Norms: [key standards]
- Safeguards: [constraint count] constraints defined
```

Then include this user-driven handoff:

```markdown
🔗 Next step: Run implementation explicitly:
   /spdd-generate @spdd/prompt/<filename>.md
```

This prompt does not implement code. Pi does not auto-chain prompt templates; the next workflow must be invoked explicitly in the conversation.

## Guardrails

- Do not proceed without input context.
- Do not implement code in this prompt. This prompt only creates the saved REASONS Canvas artifact and reports the explicit next command for the later implementation workflow.
- Do not modify source code or existing project files while generating the REASONS Canvas prompt.
- Do not modify existing files, except when explicitly writing the new prompt artifact under `spdd/prompt/`.
- Do not skip codebase exploration.
- Do not exhaustively read the whole codebase.
- Do not assume project structure without reading actual files.
- Do not hardcode stack-specific assumptions; detect the stack first.
- Do not generate generic boilerplate detached from the requirement and codebase.
- Do not leave placeholders, TODOs, or unresolved template variables in the saved prompt.
- Do not include language-specific fenced code blocks in the saved prompt; Mermaid diagrams are the only allowed fenced blocks.
- Ensure every referenced source ends up in the consolidated context, either via Pi attachment/inlining or explicit reading.
- Never proceed with partial referenced content.
- Preserve the original intent of the input context.
- Make all seven REASONS sections fully populated and mutually consistent.
- Operations must be specific and executable, including file paths/signatures when safely inferable.
- Respect existing implementations and avoid unnecessary refactoring.
- Do not force inheritance, interfaces, wrappers, layers, services, diagrams, or abstractions where they are not idiomatic or necessary for this project.
- Surface unresolved ambiguities as explicit implementation decision points with a recommended default rather than silently inventing facts.
- Prefer project-specific conventions discovered from code over generic best practices.
- Include test, validation, error-handling, and safeguard coverage for every explicit acceptance criterion or derived expectation, using `AC-n` / `DE-n` tags where applicable.

## Separation from SPDD Analysis

| Concern | SPDD Analysis Phase | REASONS Canvas Phase |
|---------|---------------------|----------------------|
| Thinking level | Strategic — what and why | Tactical — how |
| Domain | Conceptual identification | Detailed entity/data modeling |
| Solution | Direction and trade-offs | Concrete design and architecture |
| Implementation | Out of scope | Specific operations and tasks |
| Standards | Observed as context only | Norms and safeguards |
| Risks | Identify and surface | Resolve through tasks, constraints, and tests |
