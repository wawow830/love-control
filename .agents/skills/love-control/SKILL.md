---
name: love-control
description: Automate LÖVE2D games on Hyprland/Wayland using screenshot OCR, color detection, and simulated input. Use when user mentions love-control, LÖVE2D automation, game botting, clicking or reading a LÖVE window, or interacting with a love game programmatically.
---

# love-control — Agent Playbook

Automates a running LÖVE2D game window on Hyprland/Wayland. Every command prints JSON to stdout. On failure: `{"ok": false, "error": "...", "message": "..."}`.

Requires: `ydotoold`, `tesseract`, `grim`, `hyprctl`.

---

## Agent Rules

**Rule 1 — Never calculate coordinates.**
love-control handles all coordinate math. You do not read `layout_resolver.lua`, `assets.lua`, or any source file to compute pixel positions. If you need to click something, use `click-text` or `click-color`. The only exception is keyboard shortcuts (`key`, `type`) when the game explicitly supports them.

**Rule 2 — Prefer one-liners, chain when you need control.**
Use `click-text` and `click-color` for the common case of "find something, then click it". Use `wait-chain` for "wait for X, then do Y". Fall back to `chain` only when you need to inspect intermediate state, save scope mid-pipeline, or do something the one-liners don't support. Never run `snap`, `ocr`, `find-text`, and `click` as separate shell commands.

**Rule 3 — Text first. Color only when text fails.**
Always try `click-text` first. The game screen is text-readable via OCR in most cases. Only fall back to `click-color` or `probe-colors` when a text search returns nothing and the element is confirmed to be non-textual (e.g., after `read --as-text` shows no relevant labels).

**Rule 4 — Wait or wait-chain, do not sleep.**
After clicking or pressing a key, use `wait` or `wait-chain` to poll for text state changes. Do not use `sleep`.

**Rule 5 — Scope is free performance.**
If a previous `read` showed an element in a consistent text region, set a scope with `scope set --text` or `scope set --coords` before chaining. This shrinks the screenshot/OCR region, speeds up pipelines, and reduces false matches. Always `scope clear` when leaving a context.

**Rule 6 — Assert state after every interaction.**
After a click or keypress, assert that the screen changed. Use `assert --has x` after `find-text`, or use `click --verify` / `key --verify`.

---

## Canonical Patterns

### Pattern A: Click a button by text (one-shot)
```bash
love-control click-text --text "Start" --upscale --preprocess --verify
```

If you need the exact verbose chain (e.g. to save scope or inspect intermediate state):
```bash
love-control chain \
  --step snap \
  --step "ocr --upscale --preprocess" \
  --step "find-text --text 'Start' --save-scope" \
  --step click
```

### Pattern B: When text search fails — use color
If `click-text --text "Target"` times out, the element may not have visible text. Use `probe-colors` to discover what colors are currently visible, then click by color:
```bash
love-control probe-colors --count 10 --exclude-bg
# pick a likely rgb from the output, then:
love-control click-color --color 0,204,230 --tolerance 30 --verify
```

Verbose chain equivalent (only when you need control):
```bash
love-control chain \
  --step snap \
  --step "find-color --color 255,0,0 --tolerance 30 --save-scope" \
  --step click
```

### Pattern C: Keyboard-driven interaction
When the game supports number keys or arrow keys, you may use `key` instead of clicking:
```bash
love-control chain \
  --step "key 1" \
  --step snap \
  --step "ocr --upscale --preprocess" \
  --step "wait --text 'Continue' --timeout 3000"
```

### Pattern D: Click and wait for transition
```bash
love-control click-text --text "Start" --upscale --preprocess --verify
love-control wait --text "Main Screen" --timeout 10000
```

Or as a single atomic operation with `wait-chain`:
```bash
love-control wait-chain \
  --wait-text "Start" \
  --timeout 10000 \
  --step "click-text --text Start --upscale --preprocess"
```

### Pattern E: When text fails, discover colors, then click
If OCR cannot identify text targets, discover dominant colors on screen and test them:
```bash
love-control probe-colors --count 10 --exclude-bg
# inspect the JSON output, then click the dominant color:
love-control click-color --color 217,51,179 --tolerance 40 --verify
```

### Pattern F: Narrow to a panel, then read text inside it
```bash
love-control scope set --text "Settings" --padding 20
love-control chain \
  --step snap \
  --step ocr \
  --step "find-text --text Back" \
  --step click
love-control scope clear
```

---

## Playtest Workflow Template

Use this exact sequence when asked to playtest a LÖVE game:

1. **Ensure the binary is running and responsive**
   ```bash
   love-control window
   ```
   If this fails, fix the `love-control` script or start the game (`love . &`).

2. **Clear any stale scope**
   ```bash
   love-control scope clear
   ```

3. **Read the title screen**
   ```bash
   love-control read --as-text --preprocess
   ```
   Identify buttons: "Start", "Options", "Quit".

4. **Click into the game**
   ```bash
   love-control click-text --text "Start" --upscale --preprocess --verify
   love-control wait --text "Main Screen" --timeout 15000
   ```
   Or as a single atomic `wait-chain`:
   ```bash
   love-control wait-chain \
     --wait-text "Start" \
     --timeout 15000 \
     --step "click-text --text Start --upscale --preprocess"
   ```

5. **At each decision point**
   - Read the screen: `love-control read --as-text --preprocess`
   - If text labels are present ("OK", "Next", names), click them with `click-text`
   - If text search fails and the element is not detected by OCR, discover visible colors:
     ```bash
     love-control probe-colors --count 10 --exclude-bg
     # inspect the output; pick a dominant color and click it
     love-control click-color --color 0,204,230 --tolerance 30 --verify
     ```
   - If you need to click text inside a panel: set scope to the panel text, then `click-text` the option name.

6. **Verify every action**
   After every click or keypress, chain one more step:
   ```bash
   --step snap \
   --step "ocr --upscale --preprocess" \
   --step "assert --has text"
   ```

7. **End of run / state change detection**
   ```bash
   love-control wait --text "Success" --timeout 60000
   # or
   love-control wait --text "Game Over" --timeout 60000
   ```

---

## When Text Search Fails: Discovering Colors

If `click-text` and `read` cannot locate an element, it may lack visible text. You have two paths: read the source code for color constants, or probe the screen directly.

### Read source code for colors

```bash
# Common locations for color definitions
rg "colors\s*=" src/render/assets.lua
rg "color\s*=" src/render/*.lua
rg "\{[\s\d.,]+1\}" src/render/*.lua | head -20
```

Convert LÖVE float RGB to 8-bit by multiplying by 255 and rounding:
- `{0.0, 0.8, 0.9, 1}` → `0,204,230`
- `{0.9, 0.85, 0.2, 1}` → `230,217,51`
- `{0.85, 0.2, 0.7, 1}` → `217,51,179`

Pass these directly to `find-color --color` or `click-color --color`.

### Probe the screen when source is unavailable

When `assets.lua` does not exist or the colors are dynamic, use `probe-colors` to discover them from the current screen:

```bash
love-control probe-colors --count 10 --exclude-bg
```

Then use the returned `rgb` values directly in `click-color` or `find-color`.

---

## Coordinate Philosophy (Read Once, Then Forget)

- love-control operates in **window-relative screenshot pixel space**.
- `find-text` and `find-color` output `(x, y)` in that space.
- `click`, `move`, and `drag` consume `(x, y)` in that exact same space.
- love-control internally translates to `ydotool` screen coordinates. **You never touch `ydotool_factor`, `scale`, or monitor geometry.**

If you ever find yourself doing arithmetic with `cell_size`, `grid_off_x`, or `hand_gap`, you have violated Rule 1. Stop. Use `find-text` or `find-color`.

---

## Common Agent Mistakes

| Mistake | Why it fails | Correct pattern |
|---|---|---|
| Reading `layout_resolver.lua` to calculate click positions | Layout constants are logical coordinates, not screenshot pixels. Scaling, DPI, and window resize make them wrong. | `click-text` or `click-color` |
| Running `sleep 2` after a click | The game may transition faster or slower. Race conditions. | `wait --text 'Next Screen'` or `wait-chain` |
| Not using `--upscale` on small text | OCR misses 8–12px game fonts. | Always add `--upscale` to `click-text`, `read`, and `ocr` steps |
| Not using `--preprocess` on noisy or colored text | Tesseract expects black-on-white document text. Game fonts with outlines, gradients, or anti-aliasing become garbled. | Add `--preprocess` (and try `--psm 6` or `--psm 11`) when `--upscale` alone fails |
| Using `find-text` on colored shapes | OCR reads colored shapes as random characters (`\|e`, `a`, `=`). | Use `click-color` or `find-color` with known RGB values |
| Not asserting after interactions | Silent failures: the click missed, the game is still on the old screen, and the agent proceeds blindly. | Append `assert --has x` or use `--verify` |
| Re-running commands without saving a screenshot | You are debugging blind. The OCR may be failing because the screen is black, the window is behind another, or the text is rendered in a font OCR cannot read. | Save the screenshot (`snap --path /tmp/debug.png`) and inspect it before changing any parameters. |

### When a command keeps failing

The first thing to check is what the screen actually looks like:

```bash
love-control snap --path /tmp/debug.png
```

If the screenshot is blank, the game may not have a window yet. If the text is tiny or blurry, add `--upscale`. If text is still misread or missing, add `--preprocess` (and try `--psm 6` or `--psm 11`). If the element you want is a solid color with no text, use `probe-colors` to see what colors are actually on screen. Then iterate.

---

## Convenience Commands (One-liners)

These commands exist so agents cannot accidentally assemble pipelines incorrectly. They are the **preferred** way to interact with the game. Only fall back to manual `chain` when you need to inspect intermediate state.

### `click-text` — Click by text
Runs `snap → ocr → find-text → click` internally, with optional polling.

```bash
love-control click-text --text "Start" --upscale --preprocess --verify
```

| Flag | Description |
|------|-------------|
| `--text TEXT` | **Required.** Substring to search for |
| `--exact` | Exact match |
| `--upscale` | Pass through to OCR |
| `--preprocess` | Binarize/denoise before OCR |
| `--psm N` | Tesseract page mode (6=block, 11=sparse, 13=raw) |
| `--conf-min N` | Pass through to OCR |
| `--save-scope` | Save found bbox as active scope |
| `--padding N` | Padding for `--save-scope` |
| `--timeout N` | Wait up to N ms for text to appear before failing (default 5000) |
| `--verify` | Screenshot before/after; returns `"changed"` |

### `click-color` — Click by color
Runs `snap → find-color → click` internally.

```bash
love-control click-color --color 255,0,0 --tolerance 30 --verify
```

| Flag | Description |
|------|-------------|
| `--color r,g,b` | **Required.** Target RGB |
| `--tolerance N` | Color distance tolerance (default 30) |
| `--all` | Find all matches; click the first one |
| `--index N` | Click the N-th match when `--all` is used |
| `--save-scope` | Save found pixel + padding as active scope |
| `--padding N` | Padding for `--save-scope` |
| `--timeout N` | Wait up to N ms for color to appear before failing (default 5000) |
| `--verify` | Screenshot before/after; returns `"changed"` |

### `probe-colors` — Discover dominant colors
Outputs the most common non-background colors in the current screenshot, so agents can find RGB values even when `assets.lua` is buried or obfuscated.

```bash
love-control probe-colors --count 10 --exclude-bg
```

| Flag | Description |
|------|-------------|
| `--count N` | Number of colors to return (default 10) |
| `--exclude-bg` | Ignore near-black and near-white pixels |
| `--scope` | Restrict to active scope region |
| `--tolerance N` | Merge similar colors within distance N (default 30) |

**Example output:**
```json
{
  "ok": true,
  "colors": [
    {"rgb": [230, 217, 51],  "hex": "e6d933", "pixels": 1240},
    {"rgb": [0, 204, 230],   "hex": "00cce6", "pixels": 980},
    {"rgb": [217, 51, 179],  "hex": "d933b3", "pixels": 720}
  ]
}
```

**Rationale:** When `assets.lua` does not exist or uses dynamic colors, agents fall back to eyeballing screenshots. `probe-colors` gives them a data-driven discovery path.

### `wait-chain` — Wait, then act
Polls for a condition, then atomically runs a chain. Eliminates `sleep` usage.

```bash
love-control wait-chain \
  --wait-text "Ready" \
  --timeout 15000 \
  --step "click-color --color 255,0,0 --tolerance 30"
```

| Flag | Description |
|------|-------------|
| `--wait-text TEXT` | Wait for this text to appear |
| `--wait-color r,g,b` | Wait for this color to appear |
| `--tolerance N` | Color tolerance for `--wait-color` |
| `--timeout N` | Timeout in ms (default 5000) |
| `--step STEP` | Chain step to run after condition is met (repeatable) |
| `--upscale` | Upscale for OCR when using `--wait-text` |
| `--preprocess` | Binarize/denoise for OCR when using `--wait-text` |
| `--psm N` | Tesseract page mode for OCR when using `--wait-text` |
| `--conf-min N` | Minimum OCR confidence |
| `--no-scope` | Ignore active scope while polling |

---

## Atomic Commands

### `window`
Find the LÖVE window and output full geometry metadata. Other commands call this automatically if window info is missing from stdin.

**Output fields:**
| Field | Description |
|-------|-------------|
| `running` | `true` on success |
| `pid` | LÖVE process ID |
| `title` | Window title |
| `address` | Hyprland window address |
| `monitor` | Monitor ID |
| `workspace` | Workspace ID |
| `focused` | Whether window is currently focused |
| `geometry` | Logical position/size: `{x, y, w, h}` |
| `screenshot` | Physical dimensions: `{w, h}` (scale applied) |
| `scale` | Monitor scale factor |
| `ydotool_factor` | Coordinate conversion factor |

### `focus`
Focus the LÖVE window.

### `snap`
Screenshot the LÖVE window (or scoped region).

| Flag | Description |
|------|-------------|
| `--path PATH` | Save to path (default: temp file) |
| `--no-scope` | Ignore active scope |

**Output adds:** `path`, `shot` (screenshot bbox: `{x, y, w, h}`), `scale`

### `ocr`
Run OCR on an image.

| Flag | Description |
|------|-------------|
| `--path PATH` | Image path (or reads `path` from stdin) |
| `--upscale` | 2x before OCR for small text |
| `--preprocess` | Binarize/denoise before OCR |
| `--psm N` | Page segmentation mode (6=block, 7=line, 11=sparse, 13=raw) |
| `--conf-min N` | Minimum confidence (0–100) |

**Output adds:** `words` array, `path` (the image that was OCR'd)

### `find-text`
Find coordinates of text inside an OCR `words` array.

| Flag | Description |
|------|-------------|
| `--text TEXT` | **Required.** Substring to search for |
| `--exact` | Require exact token match |
| `--all` | Return every match instead of top-left |
| `--conf-min N` | Minimum confidence filter |
| `--save-scope` | Save found bbox + padding as active scope |
| `--padding N` | Padding for `--save-scope` (default 10) |

**Output adds (single match):** `x`, `y`, `text`, `conf`, `w`, `h` (and `scope` if `--save-scope`)

**Output adds (`--all`):** `matches` array, where each entry contains `x`, `y`, `text`, `conf`, `w`, `h` (and `scope` if `--save-scope`)

### `find-color`
Find coordinates of a color in an image.

| Flag | Description |
|------|-------------|
| `--color r,g,b` | **Required.** Target RGB |
| `--tolerance N` | Color distance tolerance (0–255) |
| `--path PATH` | Image path (or reads `path` from stdin) |
| `--save-scope` | Save found pixel + padding as active scope |
| `--padding N` | Padding for `--save-scope` (default 10) |

**Output adds:** `x`, `y`, `color`

### `lines`
Group OCR words into lines.

| Flag | Description |
|------|-------------|
| `--as-text` | Return a single plain-text string |
| `--compact` | Return array of line strings |

**Input requires:** `words` from stdin

**Output adds:** `lines` array. By default also preserves `words`. Use `--as-text` to get `text` string, or `--compact` for `lines` only.

### `move`
Move cursor to window-relative coordinates.

| Flag | Description |
|------|-------------|
| `--pos x,y` | Coordinates (or reads `x`/`y` from stdin) |
| `--relative` | Treat as scope-relative |

**Output adds:** `x`, `y`, `hypr` (screen coords), `ydotool` (device coords)

### `click`
Click at window-relative coordinates.

| Flag | Description |
|------|-------------|
| `--pos x,y` | Coordinates (or reads `x`/`y` from stdin) |
| `--relative` | Treat as scope-relative |
| `--button left\|right\|middle` | Default: left |
| `--duration N` | Hold duration in ms |
| `--offset x,y` | Offset from target center |
| `--verify` | Screenshot before/after; returns `"changed"` bool |

**Output adds:** `x`, `y`, `hypr` (screen coords), `ydotool` (device coords). With `--verify`, also `changed` bool.

### `drag`
Drag from one point to another.

| Flag | Description |
|------|-------------|
| `--from x,y` | Start position |
| `--to x,y` | End position (or reads `x`/`y` from stdin if `--from` is explicit) |
| `--duration N` | Duration in ms (default 500) |
| `--button left\|right\|middle` | Default: left |
| `--relative` | Treat positions as scope-relative |
| `--verify` | Screenshot before/after; returns `"changed"` |

**Output adds:** `from` (`{x, y}`), `to` (`{x, y}`). With `--verify`, also `changed` bool.

### `key`
Press keys simultaneously. LÖVE key names: `return`, `escape`, `space`, `a`, `1`, `f1`, `lshift`, `lctrl`, etc. Raw decimal (`123`) or hex (`0x1E`) keycodes accepted.

| Flag | Description |
|------|-------------|
| `keys...` | Key names |
| `--verify` | Screenshot before/after; returns `"changed"` |

**Output adds:** `keys` array. With `--verify`, also `changed` bool.

### `type`
Type literal text via ydotool.

| Argument | Description |
|----------|-------------|
| `text` | Text to type |

### `scope`
Manage the active scope.

```bash
love-control scope get          # Query active scope
love-control scope clear        # Clear it
love-control scope set --coords "10 10 100 100"
love-control scope set --text "Menu" --padding 20
love-control scope set --color 255,0,0 --tolerance 10
# Or pipe in x,y,w,h from a previous find-text/find-color:
love-control find-text --text "Menu" --save-scope | love-control scope set
```

| Flag/Arg | Description |
|----------|-------------|
| `get|set|clear` | Positional action (default: `get`) |
| `--coords x1 y1 x2 y2` | Manual rectangle |
| `--text TEXT` | Auto-detect from OCR text bbox |
| `--exact-text` | Exact match for auto-detect |
| `--color r,g,b` | Auto-detect from first matching pixel |
| `--tolerance N` | Color tolerance |
| `--padding N` | Padding around detected bbox (default 10) |
| `--upscale` | Upscale for OCR auto-detect |
| `--preprocess` | Binarize/denoise before OCR auto-detect |
| `--psm N` | Tesseract page mode for auto-detect |
| `--conf-min N` | Minimum OCR confidence |

**`scope set` stdin fallback:** If no `--coords`, `--text`, or `--color` is provided, reads `x`, `y`, `w`, `h` from stdin.

---

## Composite / Convenience Commands

### `read`
Shortcut for `snap | ocr | lines`. Returns OCR text in one shot.

| Flag | Description |
|------|-------------|
| `--no-scope` | Ignore active scope |
| `--upscale` | 2x before OCR |
| `--preprocess` | Binarize/denoise before OCR |
| `--psm N` | Tesseract page mode |
| `--as-text` | Return single plain-text string |

**Output adds:** `lines` (array), `words`, `path`, `shot`, `scale`. With `--as-text`, also `text` string.

### `click-text`
**Preferred** one-liner for clicking text. Runs `snap → ocr → find-text → click` internally. See full flag table in the Convenience Commands (One-liners) section above.

```bash
love-control click-text --text "Start" --upscale --preprocess --verify
```

### `click-color`
**Preferred** one-liner for clicking colored elements. Runs `snap → find-color → click` internally. See full flag table in the Convenience Commands (One-liners) section above.

```bash
love-control click-color --color 255,0,0 --tolerance 30 --verify
```

### `probe-colors`
Discover dominant colors on the current screen. See full flag table in the Convenience Commands (One-liners) section above.

```bash
love-control probe-colors --count 10 --exclude-bg
```

### `chain`
Run a sequence of commands **in-process**, passing JSON state between them. No shell pipe overhead, and window geometry is reused. Use this only when the convenience one-liners don't give you enough control.

By default, `chain` automatically finds and focuses the LÖVE window before executing the first step. Use `--no-focus` to skip this.

```bash
love-control chain \
  --step snap \
  --step "ocr --upscale --preprocess" \
  --step "find-text --text Start" \
  --step click \
  --step "key return"
```

| Flag | Description |
|------|-------------|
| `--step "COMMAND ARGS"` | Repeatable. One step per flag. |
| `--no-focus` | Skip the automatic window focus at chain start. |

### `wait`
Poll until a condition is met or timeout (default 5000ms).

**Generic chain polling:**
```bash
love-control wait --timeout 10000 \
  --step snap \
  --step "ocr --upscale --preprocess" \
  --step "find-text --text 'Please Wait'"
```

**Convenience shorthands:**
```bash
love-control wait --text "Game Over" --timeout 10000
love-control wait --color 255,0,0 --tolerance 10
```

| Flag | Description |
|------|-------------|
| `--timeout N` | Timeout in ms |
| `--step STEP` | Chain step to poll (repeatable) |
| `--text TEXT` | Wait for OCR text |
| `--exact-text` | Exact match |
| `--color r,g,b` | Wait for pixel color |
| `--tolerance N` | Color tolerance |
| `--no-scope` | Ignore scope |
| `--upscale` | Upscale for OCR |
| `--preprocess` | Binarize/denoise for OCR |
| `--psm N` | Tesseract page mode for OCR |
| `--conf-min N` | Minimum confidence |

### `wait-chain`
Wait for a condition, then atomically run a chain. The cleanest way to do "wait for X, then do Y".

```bash
love-control wait-chain \
  --wait-text "Combat Start" \
  --timeout 15000 \
  --step "click-color --color 255,0,0 --tolerance 30"
```

**`wait-chain` output:** `ok`, `waited` bool

| Flag | Description |
|------|-------------|
| `--wait-text TEXT` | Wait for this text to appear |
| `--wait-color r,g,b` | Wait for this color to appear |
| `--tolerance N` | Color tolerance for `--wait-color` |
| `--timeout N` | Timeout in ms (default 5000) |
| `--step STEP` | Chain step to run after condition is met (repeatable) |
| `--upscale` | Upscale for OCR when using `--wait-text` |
| `--preprocess` | Binarize/denoise for OCR when using `--wait-text` |
| `--psm N` | Tesseract page mode for OCR when using `--wait-text` |
| `--conf-min N` | Minimum OCR confidence |
| `--no-scope` | Ignore active scope while polling |

### `assert`
Fail the pipeline if expected fields/values are missing.

```bash
love-control snap | love-control ocr | love-control find-text --text "Start" | love-control assert --has x
love-control window | love-control assert --eq focused=True
```

**`assert` output:** `ok`, `assert` bool

| Flag | Description |
|------|-------------|
| `--has FIELD` | Require field existence (repeatable) |
| `--eq FIELD=VALUE` | Require exact value (repeatable) |

---

## Global Flags

| Flag | Description |
|------|-------------|
| `--raw FIELD` | Output only the named field as plain text (useful for shell scripting) |

Example:
```bash
X=$(love-control find-text --text Start --raw x)
```

---

## Exit Codes

| Code | Meaning |
|------|--------|
| 0 | Success |
| 1 | General error |
| 2 | No LÖVE window |
| 3 | Target not found |
| 4 | Command failed |
| 5 | Timeout |
| 6 | Bad arguments / missing stdin fields |

---

## Legacy Examples

**Click a button by its text:**
```bash
love-control snap | love-control ocr | love-control find-text --text "Start" | love-control click
```

**Click a button and wait for the result:**
```bash
love-control chain \
  --step snap \
  --step "ocr --upscale --preprocess" \
  --step "find-text --text 'Options'" \
  --step click \
  --step snap \
  --step "ocr --upscale --preprocess" \
  --step "assert --has text"
```

**Chain with scope narrowing:**
```bash
love-control scope set --text "Menu" --padding 15
love-control chain \
  --step snap \
  --step ocr \
  --step "find-text --text Back" \
  --step click
```

**Wait for a screen to disappear:**
```bash
love-control wait --timeout 30000 --step snap --step "find-text --text 'Please Wait'"
# Then continue...
```

**Drag an item:**
```bash
love-control chain \
  --step snap \
  --step ocr \
  --step "find-text --text 'Item Name' --save-scope" \
  --step "drag --from 100,100 --relative"
```
