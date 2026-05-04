# SPDD Prompts for Pi

Project-local [Pi coding agent](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent) prompt templates for practicing **Structured Prompt-Driven Development (SPDD)**.

SPDD treats prompts as first-class delivery artifacts: structured, version-controlled specifications that capture requirements, domain language, design intent, constraints, and task breakdown before code generation. These prompts adapt the core OpenSPDD workflow for Pi.

## Included prompts

| Pi command | Purpose | Output / effect |
|---|---|---|
| `/spdd-analysis` | Analyze requirements against the current codebase at a strategic level. | Creates `spdd/analysis/*.md` |
| `/spdd-reasons-canvas` | Convert analysis or requirements into an implementation-ready REASONS Canvas. | Creates `spdd/prompt/*.md` |
| `/spdd-generate` | Implement code from a saved REASONS Canvas, following Operations, Norms, and Safeguards. | Modifies project code |
| `/spdd-prompt-update` | Update a saved Canvas when requirements, design, or constraints change. | Updates `spdd/prompt/*.md` |
| `/spdd-sync` | Sync accepted code-side changes back into the Canvas. | Updates `spdd/prompt/*.md` |

## Workflow

```text
Requirement
  → /spdd-analysis
  → /spdd-reasons-canvas
  → /spdd-generate
  → review/test
  → /spdd-prompt-update or /spdd-sync when reality diverges
```

Core rule:

> When reality diverges, fix the prompt first — then update the code.

## Pi requirements

- Pi with prompt templates enabled.
- These files installed in `.pi/prompts/` for project-local use, or copied to `~/.pi/agent/prompts/` for global use.
- Core Pi coding tools: `read`, `bash`, `edit`, and `write`.
- Recommended Pi package: [`pi-web-access`](https://github.com/nicobailon/pi-web-access), which provides research/fetching tools used or referenced by these prompts, including `web_search`, `code_search`, `fetch_content`, and `get_search_content`.
- Recommended skill: `librarian` from `pi-web-access`, used when third-party library internals or source-backed API details are needed.

The SPDD loop is primarily local-codebase driven, but the prompts deliberately reference these web-access tools/skills for framework, library, and API questions. Pi loads prompt templates from `.pi/prompts/*.md` non-recursively. The command name is the filename without `.md`.

## Install for another Pi project

Project-local install:

```bash
mkdir -p /path/to/project/.pi/prompts
cp .pi/prompts/spdd-*.md /path/to/project/.pi/prompts/
```

Global install:

```bash
mkdir -p ~/.pi/agent/prompts
cp .pi/prompts/spdd-*.md ~/.pi/agent/prompts/
```

Then start Pi in the target project and type `/spdd-` to see the available commands.

## Artifact layout

Generated SPDD artifacts are intended to live beside the code:

```text
spdd/
  analysis/   # strategic analysis documents
  prompt/     # REASONS Canvas implementation contracts
```

Keep these files under version control so prompts and code evolve together.

## References

- Martin Fowler: [Structured Prompt-Driven Development](https://martinfowler.com/articles/structured-prompt-driven/)
- OpenSPDD: [github.com/gszhangwei/open-spdd](https://github.com/gszhangwei/open-spdd)
- REASONS Canvas: Requirements, Entities, Approach, Structure, Operations, Norms, Safeguards

## Provenance

These Pi prompts are based on the OpenSPDD core command templates and adapted for Pi prompt-template conventions, including `$ARGUMENTS`, project-local `.pi/prompts/` loading, Pi file references, and Pi tool usage.
