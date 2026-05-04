---
description: Analyze requirements into SPDD strategic context
argument-hint: "<requirement text, @file, path, glob, or folder>"
---

# SPDD Analysis

Analyze the business requirement input against the current codebase and produce a **strategic-level** enriched context document for later REASONS Canvas generation.

This is the SPDD analysis phase. Focus on **what** and **why**. Do **not** design detailed implementation steps or generate code.

Source inspiration:

- OpenSPDD `/spdd-analysis` command: https://github.com/gszhangwei/open-spdd/blob/v0.4.9/internal/templates/data/core/spdd-analysis.md
- Martin Fowler, "Structured-Prompt-Driven Development (SPDD)": https://martinfowler.com/articles/structured-prompt-driven/

## Input

The user's requirement input is:

```text
$ARGUMENTS
```

Input may contain:

1. Direct text describing the requirement
2. Pi `@file` references to requirement documents
3. Plain file paths, folder paths, or glob patterns
4. A combination of text and references

Note on Pi references: in Pi, `@file` is usually a UI-level file attachment. If file content is already attached or inlined in the conversation, treat it as already read; do not re-fetch it unless necessary to resolve ambiguity.

## Phase Goal

Produce an enriched context document that combines:

- the original business requirement, preserved verbatim
- domain concept identification grounded in the codebase
- strategic solution direction and trade-offs
- risk, ambiguity, edge-case, and acceptance-criteria analysis

Save the result under:

```text
spdd/analysis/
```

The output of this phase should be suitable input for a future `/spdd-reasons-canvas` phase.

## Steps

### 1. Validate and consolidate business input

If no requirement input is provided, stop and ask the user:

> Please provide the business requirement document or description. You can use text, `@file` references, plain paths, folders, or globs.

Do **not** proceed without business input.

If the consolidated input includes file paths, folder paths, globs, or `@`-style references:

- Treat any Pi-attached or already-inlined file content as already read; do not re-fetch it unnecessarily.
- If an `@file` reference is not attached or inlined, strip the leading `@` and treat the remainder as a plain path.
- For any path or glob that is not already attached or inlined, enumerate with `bash`/`find`/`rg --files` and read the relevant files.
- For folder inputs, list the folder and read all relevant requirement files inside.
- If a reference cannot be resolved or read, report the problem and ask the user for an alternative.
- Combine all referenced source contents with any direct text input.
- Preserve the full original meaning and information.
- Do not summarize or truncate the requirement input.

Before continuing, verify that every referenced source ended up in the consolidated business context, either via Pi attachment/inlining or explicit reading. Never proceed with partial referenced content.

### 2. Perform concept-driven codebase exploration

Do **not** read the entire codebase exhaustively. Use targeted exploration.

For repository-local searches, use `bash` with tools such as `rg`, `find`, and `ls` (or Pi's built-in `grep`/`find`/`ls` tools if available). Use external library/API search tools only for questions about third-party APIs, not for code in this repository.

#### 2a. Lightweight project fingerprint

Always begin with a lightweight project fingerprint:

- Detect the stack from primary dependency/build files, lock files, and obvious tool manifests.
- List top-level directories to understand project layout.
- Read obvious relevant configuration files only when it materially helps identify framework conventions, runtime settings, or validation commands.

Keep this step fast. Touch only a small number of files; do not enumerate every possible ecosystem manifest unless needed to disambiguate the project.

#### 2b. Extract search concepts from business input

From the consolidated requirement, extract:

- **Domain nouns**: likely entities or concepts, e.g. customer, bill, usage, pricing plan, quota
- **Action verbs**: likely operations, e.g. submit usage, calculate bill, export report
- **API surfaces**: endpoints, events, commands, queues, routes, or screens explicitly mentioned
- **Technical hints**: mentioned technologies, patterns, or domain-specific terms

Use these concepts as the search scope for codebase exploration.

#### 2c. Targeted schema/model exploration

Within the concept scope:

- Search database migrations, schema files, ORM models, entity classes, and type definitions for matching concepts using repository-local search (`rg`, `find`, `ls`, or equivalent).
- Read only matched files and directly relevant definitions.
- Follow foreign-key or ownership relationships one hop outward from matched concepts.
- If no matching schema/model code exists, note that explicitly.

#### 2d. Targeted code exploration

Within the concept scope:

- Search filenames, class names, functions, services, controllers, routes, handlers, tests, and modules for concept matches using repository-local search (`rg`, `find`, `ls`, or equivalent).
- Read matched files to understand existing business logic, validation, error handling, and architecture conventions.
- Follow direct dependencies one hop, but do not recursively expand indefinitely.
- Observe naming, layering, validation, error format, and test conventions from actual files.
- If the area appears greenfield, state that and rely only on framework/project conventions discovered from files.

#### 2e. Relevant existing SPDD context

If present, inspect:

```text
spdd/prompt/
spdd/analysis/
```

Read only files whose names or contents appear relevant to the extracted concepts.

#### 2f. Controlled expansion

If exploration reveals one clearly essential concept that was missing from the initial extraction, add it to the concept list and do one additional targeted search.

Do not expand beyond one extra hop. Note the boundary if important context may remain outside the explored scope.

### 3. Domain Concept Identification

Identify the business concepts involved at a conceptual level.

Do **not** drill into specific attributes, data types, method signatures, DTO shapes, annotations, SQL queries, or implementation details.

Cover:

- Which concepts already exist in the codebase?
- Which concepts are new?
- How do they relate at a business level?
- What ownership or lifecycle boundaries matter?
- What explicit and implicit business rules are involved?

If no existing concepts apply, write `_None — greenfield area._` rather than fabricating entries.

Use this structure:

```markdown
## Domain Concept Identification

### Existing Concepts from Codebase
- **[ConceptName]**: [business purpose] — [relationship to other concepts]

### New Concepts Required
- **[ConceptName]**: [business purpose] — [how it relates to existing concepts]

### Key Business Rules
- **[Rule]**: [which concepts it governs]
```

### 4. Strategic Approach and Trade-offs

Determine the high-level solution direction.

Do **not** specify implementation details like exact queries, annotations, JSON shapes, method signatures, file-by-file changes, or step-by-step implementation logic.

Cover:

- Overall approach to solving the requirement
- Existing architectural patterns and conventions to leverage
- General data/control flow direction
- Strategic design choices and trade-offs
- Alternatives considered and why they are not recommended

Use this structure:

```markdown
## Strategic Approach

### Solution Direction
- [High-level description of approach]

### Key Design Decisions
- **[Decision]**: [trade-offs] → [recommendation and rationale]

### Alternatives Considered
- **[Alternative]**: [why rejected]
```

### 5. Risk and Gap Analysis

Surface anything that could cause problems before detailed design begins.

Cover:

- Requirement ambiguities
- Implicit assumptions
- Edge cases and boundary scenarios
- Technical risks
- Data integrity, concurrency, migration, performance, security, or compatibility concerns
- Acceptance-criteria coverage

If the requirement contains no explicit acceptance criteria, omit the acceptance-criteria table and write this one-line note instead:

```markdown
No explicit ACs in source; coverage assessed against requirement statements above.
```

In that case, discuss coverage against the requirement statements in prose.

Use this structure when explicit acceptance criteria are present:

```markdown
## Risk and Gap Analysis

### Requirement Ambiguities
- **[Ambiguity]**: [what needs clarification]

### Edge Cases
- **[Scenario]**: [why it matters]

### Technical Risks
- **[Risk]**: [potential impact and mitigation direction]

### Acceptance Criteria Coverage
| AC# | Description | Addressable? | Gaps/Notes |
|-----|-------------|--------------|------------|
| [n] | [AC text] | Yes/Partial/No | [notes] |
```

### 6. Assemble the enriched context document

Create one Markdown document with this structure:

```markdown
# SPDD Analysis: [Derived Title]

## Original Business Requirement

[Complete original requirement text — unmodified]

## Domain Concept Identification

[Output from Step 3]

## Strategic Approach

[Output from Step 4]

## Risk and Gap Analysis

[Output from Step 5]
```

Do not include a separate exhaustive codebase inventory. Codebase exploration is working context that should be reflected through the concept, strategy, and risk sections.

### 7. Save the enriched context document

Create `spdd/analysis/` if it does not exist.

Derive a filename using:

```text
{TICKET}-{TIMESTAMP}-[Analysis]-{description}.md
```

Rules:

- **TICKET**: Extract a ticket/Jira-like identifier from the business context if present. Otherwise use `SPDD-XXX`.
- **TIMESTAMP**: Use current UTC time as `YYYYMMDDHHmm`. Use `bash` to obtain it, for example: `date -u +%Y%m%d%H%M`.
- **description**: Derive from the business context, kebab-case, fewer than 10 words.
- **collision handling**: If the target file already exists, append `-v2`, `-v3`, and so on rather than overwriting.

Examples:

```text
SPDD-XXX-202603131530-[Analysis]-token-usage-billing.md
ABC-169-202603131530-[Analysis]-monthly-report-export.md
```

Write the full document to:

```text
spdd/analysis/<filename>.md
```

This is the only file modification allowed by this prompt. Do not modify source code or existing project files.

### 8. Report completion

After saving, reply with:

```markdown
✅ Analysis complete. Enriched context saved to `spdd/analysis/<filename>.md`

📋 Analysis summary:
- Project type: [backend/frontend/fullstack/library/etc.]
- Existing concepts identified: [count]
- New concepts required: [count]
- Key design decisions: [count]
- Acceptance Criteria coverage: [covered]/[total, or "no explicit ACs"]
- Open questions/risks: [count]

🔗 Next step: Use this as input for REASONS Canvas generation:
   /spdd-reasons-canvas @spdd/analysis/<filename>.md
```

Do not continue automatically. Pi does not auto-chain prompt templates; the user must explicitly invoke the next workflow, for example:

```text
/spdd-reasons-canvas @spdd/analysis/<filename>.md
```

## Guardrails

- Do not proceed without business requirement input.
- Do not generate code.
- Do not modify source code.
- Do not modify existing files, except when explicitly writing the new analysis artifact under `spdd/analysis/`.
- Do not skip codebase exploration.
- Do not exhaustively read the whole codebase.
- Do not assume project structure without reading actual files.
- Do not hardcode stack-specific assumptions; detect the stack first.
- Do not include implementation-level details. Those belong in the REASONS Canvas phase.
- Do not leave placeholders or TODOs in the saved analysis.
- Preserve original requirements verbatim.
- Ensure every referenced source ends up in the consolidated context, either via Pi attachment/inlining or explicit reading.
- Never proceed with partial referenced content.
- Assess every explicit acceptance criterion found in the requirement.
- If no explicit acceptance criteria exist, say so and assess coverage against requirement statements in prose.
- Surface ambiguities instead of silently resolving them.

## Separation from REASONS Canvas

| Concern | SPDD Analysis Phase | REASONS Canvas Phase |
|---------|---------------------|----------------------|
| Thinking level | Strategic — what and why | Tactical — how |
| Domain | Conceptual identification | Detailed entity/data modeling |
| Solution | Direction and trade-offs | Concrete design and architecture |
| Implementation | Out of scope | Specific operations and tasks |
| Standards | Observed as context only | Norms and safeguards |
| Risks | Identify and surface | Resolve through tasks, constraints, and tests |
