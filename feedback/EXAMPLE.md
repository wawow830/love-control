# Example Feedback Entry

## Context
Was working on the `love-control` binary — noticed the error messages
don't consistently use the `{"ok": false, "error": "..."}` JSON format
described in the skill docs.

## Observation
In `src/main.rs` around line 142, `handle_screenshot()` returns a plain
string error instead of JSON. This means agents parsing stdout will fail
to detect the error condition properly.

## Recommendation
Change the error return in `handle_screenshot()` to use the standard
error wrapper function from `src/errors.rs`. Then add a test that
verifies JSON error output.

## Priority
Medium

## Labels
bug, technical-debt

## Related
- `src/main.rs:142`
- `src/errors.rs`
- Skill doc: `.agents/skills/love-control/SKILL.md`

---
_Agent: pi (example)_
_Date: 2026-05-26_
