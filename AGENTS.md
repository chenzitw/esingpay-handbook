# Agents

Read this file first when working on this handbook. It captures the working norms and navigation map every AI agent needs before producing output.

## Working norms

### Conversation rhythm

- Concise over exhaustive. Keep messages, options, and proposals lean; don't scaffold work the human collaborator should decide.
- One focused question per turn, with a stated preference and rationale.
- Do not require explicit validation to surface concerns. If something looks off, say so on the next turn.
- Pushback is short and pointed. A reasonable disagreement is not a failure; convergence by capitulation is.

### Surfacing options before converging

- Present divergent options before recommending one. For each option, state cost and tradeoff briefly; then pick a preference explicitly.
- When sharing a position intended for forwarding to another agent (cross-AI peer review), prefix with the speaker name in third person — "Claude thinks…", "ChatGPT suggests…", "Gemini argues…" — so the source is unambiguous.

### Incremental delivery with explicit gates

For multi-phase or multi-decision work:

1. Propose the phase or decision breakdown first.
2. Wait for approval or redirect before proceeding.
3. Expand the approved phase in detail.
4. Wait for confirmation.
5. Write confirmed content into the artifact.

Do not redesign or challenge frozen decisions mid-implementation. Flag scope boundaries proactively before crossing them.

### Document authoring

- Specification documents must be independently readable by AI implementers. Write for them, not as internal notes.
- Production order: leaf files before parent files (fine-to-coarse).
- Strip heading numbers from all handbook documents and from each codebase's `guide/` documents.
- Default to minimal. Don't add helpers, examples, or scaffolding "to be safe" — wait until requested.

### Reference baselines

When implementing or specifying a feature that mirrors existing functionality:

- Surface unreasonable aspects of the baseline rather than blindly replicating them.
- Propose cross-plan sync amendments where the mirror reveals shared issues.
- Flag items for an "improve existing feature" backlog when divergence isn't worth absorbing now.

Mirror baseline is engineering leverage, not spec authority.

## Where to look

| Need                                               | Location                                           |
| -------------------------------------------------- | -------------------------------------------------- |
| Workflow framework (PSDD overview)                 | `convention/workflow.md`                           |
| Writing a proposal / design / blueprint / plan     | `convention/workflow-<tier>.md`                    |
| Version cycle / closing / hygiene / distillation   | `convention/workflow-closing.md`                   |
| Version status across versions                     | `VERSIONS.md`                                      |
| Current version scope / backlog / spec persistence | `idea/vX.Y/README.md`                              |
| Specific feature within a version                  | `idea/vX.Y/<topic>-*.md`, `idea/vX.Y/<topic>/*.md` |
| Settled, cross-version architectural specs         | `spec/`                                            |
| Codebase architecture rules                        | `<codebase>/guide/`                                |
| Codebase-specific agent setup                      | `<codebase>/AGENTS.md`                             |

Consult this table before exhaustive search. The handbook is designed for minimal-read navigation.

## Reading discipline

- For workflow questions, go to `convention/` first.
- For codebase facts (existing schema, file paths, patterns), request a survey from a CLI-side agent rather than guessing.
- When a project convention is unfamiliar, read the relevant `convention/*.md` before producing output.

## Maintenance

Codex maintains the "Where to look" table. Other agents may flag entries needing updates but should not rewrite it directly. The rest of this file is updated as collaboration norms evolve; propose changes explicitly rather than editing inline.
