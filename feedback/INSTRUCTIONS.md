# feedback/ — Agent Feedback Directory

This directory is where agents leave structured feedback, observations, and
notes about their work in this project.

## Purpose

Agents working in this repository SHOULD use this directory to:
- Log observations about code quality, architecture, or patterns they notice
- Leave suggestions for future improvements
- Record context about decisions made during their session
- Note recurring issues or anti-patterns they encounter
- Document anything the good boy (or another agent) should know later

## File Format

Each feedback entry is a single Markdown file in the root of `feedback/` with
the following naming convention:

```
YYYY-MM-DD--short-descriptive-slug.md
```

For example:
- `2026-05-26--refactor-too-many-params.md`
- `2026-05-26--suggestion-add-progress-bar.md`
- `2026-05-26--found-dead-code.md`

### Required Sections

Every feedback file MUST include these sections in order:

```markdown
# Topic or Title

## Context
What were you working on? What triggered this observation?

## Observation
What did you see? Be specific — include file paths, line numbers, or
command output if relevant.

## Recommendation
What should be done about it? Concrete action items are preferred.
```

### Optional Sections

```markdown
## Priority
High / Medium / Low

## Labels
bug, suggestion, question, praise, technical-debt, refactor, documentation

## Related
Links to related files, commits, PRs, or issues.
```

## Rules for Agents

1. **Append, never overwrite.** Each entry is a new file. Do not edit or
   delete existing feedback entries unless explicitly instructed.

2. **One observation per file.** If you have multiple unrelated observations,
   create multiple files.

3. **Be specific.** Vague feedback helps no one. Include file paths, line
   numbers, commit hashes, or reproduction steps when applicable.

4. **No personal notes.** This is for actionable project feedback, not
   internal agent reasoning or logs.

5. **Sign with agent name + date** in a `---` footer block at the bottom
   of each file.
