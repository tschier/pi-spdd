---
description: Generate or update implementation code from a saved SPDD REASONS Canvas prompt, strictly following Operations, Norms, and Safeguards
argument-hint: "<@spdd/prompt/file.md or path to REASONS Canvas prompt>"
---

# SPDD Generate

Generate implementation code from a structured SPDD prompt file using the **REASONS Canvas** contract:

- **R** — Requirements
- **E** — Entities
- **A** — Approach
- **S** — Structure
- **O** — Operations
- **N** — Norms
- **S** — Safeguards

This is the implementation phase of the SPDD workflow. Read the saved Canvas, validate it, then implement exactly what its Operations define while applying its Norms and enforcing its Safeguards.

Source inspiration:

- OpenSPDD `/spdd-generate` command: https://github.com/gszhangwei/open-spdd/blob/v0.4.9/internal/templates/data/core/spdd-generate.md
- Martin Fowler, "Structured-Prompt-Driven Development (SPDD)": https://martinfowler.com/articles/structured-prompt-driven/

Core SPDD principle:

> When reality diverges, fix the prompt first — then update the code.

## Input

The structured prompt file is:

```text
$ARGUMENTS
```

Input should be a saved REASONS Canvas prompt, usually under:

```text
spdd/prompt/
```

Input may contain:

1. A Pi `@file` reference to a prompt file
2. A plain file path
3. A glob that resolves to exactly one prompt file

Note on pi references: in pi, `@file` is usually a UI-level file attachment. If file content is already attached or inlined in the conversation, treat it as already read; do not re-fetch it unless necessary to resolve ambiguity.

## Phase Goal

Create or update implementation files so the codebase conforms to the saved REASONS Canvas.

The structured prompt is the contract between design and implementation. Generated code must correspond one-to-one with the Canvas, especially the **Operations**, **Norms**, and **Safeguards** sections.

## Phase Boundary

`/spdd-generate` owns only the Canvas → code implementation phase.

It may:

- read and validate a saved REASONS Canvas
- implement Operations in order
- fix implementation mistakes where the Canvas is clear
- run validation and report results

It must not:

- update the saved Canvas to change requirements or design
- sync code-side refactors back into the Canvas
- invent or inline `/spdd-prompt-update` or `/spdd-sync` behavior

If the Canvas needs to change, stop and hand off to `/spdd-prompt-update` once available, or ask the user to update the Canvas explicitly. If code has changed independently and the Canvas needs to be synchronized, stop and hand off to `/spdd-sync` once available. Pi does not auto-chain prompt templates.

## Steps

### 1. Validate and read the structured prompt

If no input is provided, stop and ask the user:

> Please provide the path to the structured prompt file, for example `@spdd/prompt/SPDD-XXX-202603131530-[Feat]-api-user-registration.md`.

Do **not** proceed without a valid structured prompt.

Resolve the input:

- Treat already attached or inlined `@file` content as read.
- If the argument starts with `@` but no file is attached or inlined, strip the leading `@` and treat the remainder as a plain path.
- For a plain path, use `read` to read the entire file.
- For a glob, use `bash` to resolve it. Continue only if it resolves to exactly one file; otherwise ask the user to choose.
- If the file cannot be read, report the problem and ask for a valid path.

Read the entire prompt file carefully. Do not implement from a partial read.

### 2. Parse the REASONS Canvas

Extract and understand these sections:

| Section | Purpose | Implementation usage |
|---------|---------|----------------------|
| **Requirements** | Overall goal, scope, DoD, AC/DE IDs | Determine intended behavior and acceptance verification |
| **Entities** | Domain model, data shapes, relationships | Guide data modeling and API/type/class design |
| **Approach** | Implementation strategy and trade-offs | Guide architectural choices |
| **Structure** | Components, files, dependencies, data flow | Verify layering and file/component boundaries |
| **Operations** | Concrete implementation tasks | Execute in the defined order |
| **Norms** | Engineering standards and project conventions | Apply to all code changes |
| **Safeguards** | Non-negotiable constraints | Enforce strictly |

If any required section is missing, empty, contradictory, or unusably vague, stop before code changes and report the issue as a prompt defect. Suggest the smallest Canvas update needed.

### 3. Analyze current project context

Before changing code, perform targeted codebase exploration. Do **not** read the whole repository exhaustively.

#### 3a. Lightweight project fingerprint (only if not implied by the Canvas)

If the Canvas's Structure section already states concrete file paths and the project's stack is unambiguous from those paths, keep this step brief. Otherwise, identify the stack and tooling by reading the primary project files when present:

- JS/TS: `package.json`; note lock files such as `pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`, or `bun.lockb`
- Python: `pyproject.toml`, `requirements.txt`, `Pipfile`
- Go: `go.mod`
- Rust: `Cargo.toml`
- Ruby: `Gemfile`
- PHP: `composer.json`
- Java/Kotlin: `pom.xml`, `build.gradle`, `build.gradle.kts`
- Elixir: `mix.exs`
- Deno: `deno.json`, `deno.jsonc`
- Nix: `flake.nix`

Also list top-level directories and read one obvious relevant config file when useful.

#### 3b. Locate existing patterns

Using file paths and concepts from the Canvas:

- Read every existing file explicitly named in Operations or Structure.
- Search for nearby or similar controllers, services, routes, handlers, components, migrations, models, validators, tests, fixtures, and utilities.
- Read only directly relevant pattern files and one-hop dependencies needed to implement safely.
- Observe naming, formatting, layering, validation, error handling, dependency injection, logging, testing, and command conventions.

Use repository-local tools (`bash` with `rg`, `find`, `ls`, plus `read`) for local code. For questions about third-party APIs, frameworks, or libraries, prefer `code_search`, `web_search`, or the `librarian` skill instead of inventing API details from local grep results.

### 4. Validate the Operations sequence before implementation

Review the **Operations** section before editing files.

Verify:

1. **Dependency order**
   - Independent constants/types/enums/schema changes come before dependents.
   - Each operation depends only on components already existing or created by earlier operations.
   - No circular dependency is introduced.

2. **Completeness**
   - Each file/component listed in Structure is covered by Operations or explicitly unchanged.
   - Each Requirement AC/DE has at least one implementation or test operation.
   - No logical gap exists between Operations.

3. **Consistency**
   - Entities match the data shapes used by Operations.
   - Approach matches the implementation tasks.
   - Dependencies and layering match Structure.
   - Norms and Safeguards are enforceable by the planned operations.

If blocking issues are found, stop and report:

```markdown
⚠️ Structured prompt issue found before implementation.

- Issue: [description]
- Canvas section(s): [Requirements/Entities/Approach/Structure/Operations/Norms/Safeguards]
- Why this blocks generation: [reason]
- Suggested prompt update: [specific minimal change]
- To resume: update the saved Canvas first, using `/spdd-prompt-update` once available or an explicit user-approved edit to the Canvas file, then re-run `/spdd-generate <same-prompt-file>`. Pi does not auto-chain prompt templates.
```

Do **not** silently re-plan the sequence. The Operations order is the designed execution order from the Abstraction phase.

### 5. Implement strictly following Operations

Execute each Operation in order.

For each operation:

1. Re-read the operation specification:
   - Responsibility
   - File path(s)
   - Create/update/delete intent
   - Fields, methods, signatures, routes, commands, migrations, props, event contracts, or schema changes
   - Validation rules
   - Business logic
   - Error handling
   - AC/DE tags
   - Completion criteria

2. Apply Norms:
   - Walk the Norms in the order and numbering used by the Canvas. Apply each numbered category, treating any `N/A — …` entry as deliberately skipped.
   - Follow project style, names, formatting, layering, dependency injection, error patterns, test style, and documentation expectations.
   - Prefer existing project conventions over generic best practices.

3. Enforce Safeguards:
   - Walk the Safeguards in the order and numbering used by the Canvas. Enforce each numbered category, treating any `N/A — …` entry as deliberately skipped.
   - Preserve exact error messages, status codes, response shapes, validation semantics, security constraints, compatibility requirements, and data integrity rules.
   - Do not weaken constraints to make implementation easier.

4. Edit code:
   - Use `read` before editing existing files.
   - Use `edit` for precise changes to existing files.
   - Use `write` for new files or complete file rewrites when appropriate.
   - Keep changes scoped to files and behavior required by the Canvas.
   - For existing generated code, perform targeted updates rather than rewriting unrelated areas.

Targeted diffs on re-runs:

- When `/spdd-generate` is re-invoked after a Canvas update, treat existing implementation as the baseline.
- Compute the smallest set of code changes that brings the codebase back into conformance with the updated Canvas.
- Do not regenerate unaffected files. Do not reformat or restructure code outside the diff implied by the Canvas change.
- Prefer `edit` over `write` for any file that already exists.

Implementation constraints:

- Do not add features, endpoints, fields, methods, abstractions, dependencies, tests, or refactors not specified or required by the Canvas.
- Do not change public contracts except where the Canvas explicitly requires it.
- Do not change exact method signatures, field names, error messages, HTTP status codes, or data contracts specified in the Canvas.
- Do not ignore existing project patterns.
- Do not make unrelated cleanup changes.
- Do not create commits unless the user explicitly asks.

### 6. Batch validation after all implementation changes

After all Operations are implemented, run unified validation.

Choose validation commands from (a) the saved Canvas's Norms section if it lists them, (b) `package.json` scripts / `Makefile` targets / language-specific config files discovered in Step 3a, and (c) any project README/CONTRIBUTING file already read. Examples, depending on stack:

- JS/TS: package-manager lint/typecheck/test/build scripts from `package.json`
- Python: configured formatter/linter/type checker/test command, e.g. `pytest`, `ruff`, `mypy`
- Go: `go test ./...`
- Rust: `cargo test`, `cargo clippy`
- Java/Kotlin: `mvn test`, `./gradlew test`, or configured equivalent

Prefer the narrowest reliable validation first when the full suite is expensive, then run broader validation if appropriate.

Validate:

1. **Syntax/compile/type correctness**
   - Imports resolve.
   - Types match.
   - Generated files compile or parse.

2. **Tests and acceptance coverage**
   - Build a mapping from each `AC-n` / `DE-n` in Requirements to (a) the Operation(s) that implement it and (b) the test(s) or manual check that verify it.
   - Every `AC-n` / `DE-n` must appear on the implementation side and on the verification side.
   - Expected errors, messages, status codes, response fields, and edge cases match the Canvas exactly.

3. **Structure and architecture**
   - Layering and dependency direction match Structure.
   - No circular or inappropriate coupling was introduced.
   - Existing extension points were reused where required.

4. **Norms and Safeguards**
   - Coding style and test style match the project.
   - Each numbered Norm and Safeguard category is applied/enforced or explicitly marked `N/A` according to the Canvas.
   - Non-negotiable constraints remain enforced.

Fix validation failures that are implementation mistakes relative to the Canvas.

If validation reveals that the Canvas itself is wrong, incomplete, or contradictory, stop and report that the prompt must be updated first. Do not patch behavior directly around a prompt defect.

### 7. Review and iteration loop

If an issue is discovered during implementation, testing, or review, classify it:

#### A. Implementation mistake

The Canvas is clear and correct, but the code does not conform.

Action: fix the code so it matches the Canvas, then re-run relevant validation.

#### B. Prompt defect or requirement/design change

The desired behavior differs from the Canvas, or the Canvas is ambiguous/incorrect.

Action: stop and ask to update the structured prompt first. Trace the issue to the right section:

- Wrong requirement or DoD → update **Requirements**
- Missing entity, relationship, or data shape → update **Entities**
- Flawed strategy or trade-off → update **Approach**
- Incorrect file/component/dependency plan → update **Structure**
- Wrong task, method, signature, or logic detail → update **Operations**
- Missing engineering standard → update **Norms**
- Missing non-negotiable constraint → update **Safeguards**

To resume, update the saved Canvas first, using `/spdd-prompt-update` once available or an explicit user-approved edit to the Canvas file, then re-run `/spdd-generate <same-prompt-file>`. Pi does not auto-chain prompt templates.

After the prompt is updated, regenerate only the affected code.

### 8. Report completion

Reply with:

```markdown
✅ SPDD generation complete for `<prompt-file>`

📦 Files changed:
- `path/to/file` — created/updated/deleted — responsibility

🔍 Validation results:
- Compile/typecheck: pass/fail/not run — [literal command]
- Lint/format: pass/fail/not run — [literal command]
- Tests: pass/fail/not run — [literal command]
- AC/DE coverage: [covered]/[total]
- Norms: pass/fail — [numbered summary, e.g. `1 pass, 2 pass, 3 N/A`]
- Safeguards: pass/fail — [numbered summary, e.g. `1 pass, 2 pass, 3 N/A, 4 pass`]

🔍 AC/DE coverage:
| ID | Implemented in | Verified by |
|----|----------------|-------------|
| AC-1 | `path/to/file` op N | `path/to/test` or manual check |

📝 Notes:
- Deviations: [none or list]
- Assumptions: [none or list]
- Prompt issues requiring update: [none or list]
```

Each `[literal command]` value must be a literal shell command runnable from the repo root, e.g. `pnpm typecheck`, `pytest -q`, `go test ./...`. If a command was skipped, state the reason.

If validation could not be run, explain why and provide the exact command the user should run.

## Guardrails

- Do not proceed without reading the entire structured prompt file.
- Do not implement from an incomplete, missing, or contradictory Canvas.
- Do not re-plan or reorder Operations except to stop and report a prompt defect.
- Do not skip any Operation defined in the Operations section.
- Do not add extra features, endpoints, fields, abstractions, dependencies, or tests beyond the Canvas.
- Do not modify exact method signatures, field names, error messages, status codes, or contracts specified by the Canvas.
- Do not patch behavior directly when the prompt is wrong; update the prompt first.
- Do not modify the structured prompt unless the user explicitly asks for a prompt update.
- Do not modify unrelated source files.
- Do not perform broad refactors unless explicitly required by Operations.
- On re-runs, produce the minimal targeted diff implied by the Canvas; do not regenerate unchanged files.
- Do not create commits unless explicitly asked.
- Always align with existing project conventions.
- Always apply Norms to generated code.
- Always enforce Safeguards strictly.
- Always verify against acceptance criteria or derived expectations.
- Always run relevant validation when tooling is available.
- Always report validation commands and results.

## SPDD Workflow Context

This command is the implementation phase in the local SPDD loop:

```text
Requirement → /spdd-analysis → /spdd-reasons-canvas → /spdd-generate → Review/Test → /spdd-sync or /spdd-prompt-update as needed
```

The prompt and code must evolve together. If behavior changes after generation, update or sync the structured prompt so it remains the version-controlled source of implementation intent.

Note: `/spdd-generate` is generate-only. If a behavior/design change is needed, hand off to `/spdd-prompt-update` before re-running generation. If code-side refactoring needs to be reflected back into the Canvas, hand off to `/spdd-sync`. Do not inline those workflows inside `/spdd-generate`.
