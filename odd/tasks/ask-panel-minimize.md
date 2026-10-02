# Minimize the ask_user_question panel

## Objective / authorization

Give the first-party `ask_user_question` questionnaire a minimized state so a user on a short
terminal can read the transcript while the question is pending, with a persistent bar that keeps
the pending question visible, and a single toggle back to the full panel. The two preferences
that shape it (default state, minimized indicator) are configurable from `/gentle:customize`.

User authorized the change and fixed its scope: minimize only (no body ceiling, no preview
scrolling, no overlay rewrite), default state stays `expanded` so today's behavior does not
change, and both preferences are configurable in the configuration tool.

Base: `main` (= `upstream/main`, `4fcddc2f`). This change must be verifiable against upstream
alone so it can be proposed to the community; merging it into `local` for personal use is a
separate, later decision.

## Problem and constraints

The questionnaire mounts through `ctx.ui.custom()` as a native dock swap, not an overlay
(`extensions/ask-user-question.ts`, "Native dock swap, never an overlay"). The dock area is sized
by the height of the lines the component returns from `render(width)`
(`node_modules/@earendil-works/pi-tui/dist/layout.js:32` — `measureHeight`), and `render` receives
only `width`, never a height.

One question body renders one line per option label plus one line per option description, plus
the tab strip, the free-text row, a blank and the hint
(`lib/questionnaire/questionnaire-view.ts`, `renderBody` / `hint`). A four-option question with
descriptions is ~12-14 lines before the two borders the extension adds. On a 24-row terminal that
leaves the transcript at or near zero visible rows: the panel does not cover the transcript, it
consumes it.

There is no escape today other than cancelling: the questionnaire owns the whole dock while it
is open, and no part of it ever steps aside to give rows back.

Transcript scrolling is not the problem, and an earlier draft of this document was wrong about it.
The deferral gate that freezes the transcript during a question applies only to overlays:
`TuiAltScreen.shouldDeferViewportInputToOverlay()` requires `isOverlayFocused()`, true only when the
focused component is in the overlay stack (`node_modules/@earendil-works/pi-tui/dist/tui-alt-screen.js:461`,
`node_modules/@earendil-works/pi-tui/dist/tui.js:494`), and this questionnaire mounts as a plain dock
swap without `overlay: true`. So `tui.altScreen.pageUp`/`pageDown` (real defaults `pageUp`/`pageDown`),
the wheel, `home`/`end` and transcript search keep scrolling the transcript while the questionnaire has
focus. That matches the maintainer triage recorded in upstream issue #1141, where the frozen case was an
overlay-mounted dialog and "a plain dock swap scrolled normally". The problem is only how many rows are
left to scroll through.

Related, deliberately out of scope:

- #1340 (open): an option `preview` has no height budget, so a long preview grows the panel past
  the terminal. Its suggested fix is a body ceiling plus a scrolling preview pane.
- #1356 (open): per-question notes bound to a new `n` key.
- #1273 (open): pointer semantics (select on first click, confirm on a second click).

Constraints that shape the design:

- `app.tools.expand` (default `ctrl+o`) is the app's existing expand/collapse idiom
  (`node_modules/@earendil-works/pi-coding-agent/dist/core/keybindings.js:52`) and is resolvable
  from a custom component both for matching and for the displayed label
  (`KeybindingsManager.matches` / `.getKeys`, `pi-tui/dist/keybindings.d.ts:263-264`;
  `app.*` ids are type-valid through declaration merging,
  `pi-coding-agent/dist/core/keybindings.d.ts:49-52`).
- Terminal geometry is reachable from the custom-component factory as `tui.terminal.rows`
  (`pi-tui/dist/tui.d.ts:216`, `pi-tui/dist/terminal.d.ts:68`); the repo already reads it this way
  (`extensions/gentle-shell.ts`, `rowsAvailable: () => ... tui.terminal.rows ...`).
- Preferences follow the existing single-file policy pattern: one JSON file in the Gentle Pi
  config home, a versioned `schema` key, tolerant read (missing or malformed reads as defaults,
  with a `malformed` flag), and a writer that refuses to replace a malformed file
  (`lib/card-style-policy.ts`).

## Design

### Preferences

New module `lib/ask-panel-policy.ts`, mirroring `lib/card-style-policy.ts`.

```ts
export const ASK_PANEL_SCHEMA = "gentle-pi.ask-panel/v1";
// File: ask-panel.json in gentlePiConfigHome().
export const ASK_PANEL_INDICATOR = { MINIMAL: "minimal", TABBED: "tabbed", ANSWERS: "answers" } as const;
export const ASK_PANEL_DEFAULT_STATE = { EXPANDED: "expanded", AUTO: "auto", COLLAPSED: "collapsed" } as const;
export const ASK_PANEL_DEFAULTS = { indicator: "minimal", defaultState: "expanded" } as const;

export interface AskPanelPreferences { indicator: AskPanelIndicator; defaultState: AskPanelDefaultState }

export function parseAskPanelFile(raw: string): AskPanelPreferences | undefined;
export function resolveAskPanelPreferences(options?: { gentlePiConfigHome?: string }): {
  preferences: AskPanelPreferences;
  source: "global_file" | "default";
  malformed: boolean;
  globalFile: string;
};
export function writeAskPanelPreferences(preferences: AskPanelPreferences, options?: { gentlePiConfigHome?: string }): string;
```

Semantics, identical in shape to the card-style policy:

- Missing file: defaults, `source: "default"`, `malformed: false`.
- Malformed or unreadable: defaults, `source: "global_file"`, `malformed: true`.
- Malformed means: not a JSON object, an array, a missing or wrong `schema`, missing or unknown
  key set (exactly `schema`, `indicator`, `defaultState`), or a value outside its allowed set.
- `writeAskPanelPreferences` throws on an invalid argument and refuses to overwrite a malformed
  file; it writes `{ schema, indicator, defaultState }` with mode `0600` through a temporary
  file plus rename, and returns the path.

### View behavior

`QuestionnaireView` gains two options and one interaction:

```ts
export interface QuestionnaireViewOptions {
  // ...existing
  preferences?: AskPanelPreferences;   // defaults to ASK_PANEL_DEFAULTS
  terminalRows?: number;               // absent -> no automatic minimization
}
```

- New state: `collapsed` (boolean) and `userToggled` (boolean).
- `defaultState: "collapsed"` starts minimized. `"expanded"` (default) never minimizes on its own.
- `defaultState: "auto"` minimizes on render only when the expanded panel would not fit the
  terminal: compare the expanded line count against `terminalRows - RESERVED_CHROME_ROWS -
  MIN_TRACE_ROWS`, with named module constants (`RESERVED_CHROME_ROWS = 4`,
  `MIN_TRACE_ROWS = 8`, both heuristic and documented as such). The automatic decision is sticky:
  once minimized automatically it stays minimized until the user toggles, so cursor movement and
  preview lines cannot make the panel oscillate. No `terminalRows` means no automatic
  minimization.
- An explicit user toggle always wins and disables automatic minimization for that view instance.
- Toggle key: `app.tools.expand` through the injected `KeybindingsManager`, with a
  `matchesKey(data, "ctrl+o")` fallback when no manager is injected. The displayed label comes from
  `keybindings.getKeys("app.tools.expand")[0]`, falling back to `ctrl+o`.
- `escape` still cancels while minimized. `up`/`down`/`enter`/`space` while minimized must not
  commit, move the cursor, or open the editor; they are ignored so no answer can be committed
  accidentally from a minimized panel.
- Toggling minimize while the free-text editor is open closes the editor first, keeping the
  existing `customDraft` behavior, so expanding shows the body again with the draft intact.
- A left click on the minimized bar expands. Mouse handling must route the bar through
  `lineOwners` without committing anything.

### Minimized rendering

`render(width)` returns the bar while minimized, so the dock gives its rows back to the transcript.
Every line still fits `width` (reuse the view's `wrap`/`push`).

| Indicator | Lines |
| --- | --- |
| `minimal` (default) | `▸ 1/3 · Header · 4 options · ctrl+o expand · esc cancel` |
| `tabbed` | the existing tab strip (progress, answered `✓` chips), then `▸ minimized · ctrl+o expand · esc cancel` |
| `answers` | `▸ 1/3 · Header · answered: Alpha, Beta · ctrl+o expand` (`answered: none` when the active question has no answer) |

`▸`, the progress and the header chip use the same accent treatment as the expanded view
(`accent()`); hints use `dim`/`muted`. The key label in every variant is the resolved
`app.tools.expand` label. The bar's rendered lines map to a dedicated line owner (a reserved
`rowIndex`, distinct from the header/body owners) so a click toggles instead of selecting.

The expanded hint line gains the toggle hint, e.g.
`↑↓ move · enter select · tab switch · ctrl+o minimize · esc cancel`.

### Customize surface

- `lib/visual-customize-view.ts`: extend `CustomizeCategory` with `"Ask"`.
- `extensions/gentle-shell.ts`: register the new category and two rows, next to the existing
  Cards rows and using the same `add(label, notice, action, preview)` helper:

| Row label | Cycles | Preview |
| --- | --- | --- |
| `Ask panel: default state · <state>` | `expanded` → `auto` → `collapsed` | the resulting sample bar, including the `auto` note |
| `Ask panel: minimized indicator · <indicator>` | `minimal` → `tabbed` → `answers` | the sample bar for that indicator |

Activating a row writes through `writeAskPanelPreferences` and notifies
`Ask panel saved. Applies to the next questionnaire.` A malformed preferences file must surface as
a notification through the existing `onError` path, never as a silent overwrite.

### Extension wiring

`extensions/ask-user-question.ts`, TUI branch only: resolve the preferences once per invocation,
read `tui.terminal.rows` inside the `ctx.ui.custom` factory, and pass both into
`new QuestionnaireView({ ... })`. The RPC-dialog path (`askThroughDialogs`) has no panel and is
untouched.

## Non-goals

- No body height ceiling, no scrolling preview pane, no change to `MIN_PREVIEW_WIDTH` or the
  split preview (#1340 stays open).
- No overlay conversion and no `OverlayHandle.setHidden()`: converting the panel to an overlay would
  move it into the one mount shape that does freeze the transcript (the deferral gate above), trading
  a space problem for an input problem.
- No new keybinding id and no new configurable binding; the toggle reuses `app.tools.expand`.
- No change to answer semantics, schema, validation, or the RPC-dialog path.

## Tasks

- [x] T1: Add `lib/ask-panel-policy.ts` and `tests/ask-panel-policy.test.ts`, test-first.
- [x] T2: Add the minimized state, toggle, indicator variants and automatic mode to
      `lib/questionnaire/questionnaire-view.ts`, with tests in `tests/questionnaire-view.test.ts`,
      and wire the preferences and terminal rows in `extensions/ask-user-question.ts`.
- [x] T3: Expose both preferences in `/gentle:customize` (category `Ask`), with tests.
- [x] T4: Document the behavior, then run the full verification set.
- [ ] T5: Delivery in the user's fork: PR against `local` referencing issue #56, then the user
tests `local`. Proposing it upstream (`Gentleman-Programming/gentle-shell`, base `main`) is a
separate, later decision.

## Acceptance and checks

Test-first: write the failing tests first, observe RED, then implement to GREEN.

- `tests/ask-panel-policy.test.ts`: defaults on a missing file; parse round-trip; malformed cases
  (non-object, array, wrong schema, unknown key, unknown value) resolve to defaults with
  `malformed: true`; the writer refuses a malformed file and round-trips a valid one with the
  schema key and mode `0600`; `gentlePiConfigHome` injection works.
- `tests/questionnaire-view.test.ts`: minimized render is one line (`minimal`, `answers`) or two
  (`tabbed`) and never exceeds the width; the bar names the question and the toggle key; the
  toggle collapses and expands preserving cursor, toggles and committed answers; `escape` still
  cancels while minimized; `enter`/arrows/`space` while minimized commit nothing; a click on the
  bar expands; `defaultState: "collapsed"` starts minimized; `defaultState: "auto"` minimizes on a
  short terminal and does not on a tall one, and an explicit toggle overrides it; a remapped
  `app.tools.expand` binding toggles on the remapped key.
- Customize: the `Ask` category exists and both rows cycle and persist.
- Commands: `pnpm run typecheck`, the focused test files, `pnpm test`,
  `pnpm run check:runtime-modules`, `node scripts/verify-package-files.mjs`.

Honest reporting: a skipped or failed check is recorded as such.

## Delivery

`main` is the upstream mirror and carries no local work; this branch is based on it. The community
proposal is a PR against `Gentleman-Programming/gentle-shell` with base `main` and one linked
approved issue. Merging the same change into `local` for personal use is a separate decision for
the user. Nothing is pushed, opened or merged without explicit authorization.

## Progress

Created with the plan; task checkboxes are marked only from observed outcomes.

T1 + T2 (bounded writer, branch `feat/ask-panel-minimize`, base `main` = `4fcddc2f`): implemented
test-first. RED captured at assertion level for both files (T1: parse returned `undefined` and the
malformed flag/source were wrong against a signature stub; T2: 21 pass / 11 fail on the new minimized
tests). GREEN on the focused command -- `tests/ask-panel-policy.test.ts`, `tests/questionnaire-view.test.ts`,
`tests/ask-user-question.test.ts`, `tests/questionnaire-schema.test.ts` -- 73 pass, 0 fail, re-verified by
the parent as 38 pass on the two owned files. `pnpm run typecheck`: 187 recorded diagnostics, no regressions.

Deliberate, spec-silent decision recorded from that run: while minimized, `tab` is ignored along with
`up`/`down`/`enter`/`space`, so nothing but the toggle, `escape` and the bar click has an effect, and
navigation resumes on expand. `terminalRows` is read with optional chaining inside the extension factory
so the existing RPC-host fakes (no `terminal`) keep passing; production always provides it.

Discovery, pre-existing and out of scope: `Text.render("")` returns `[]`, so the view's `push("")`
separator lines never render as blank lines. The `auto` thresholds are therefore calibrated against real
rendered lines (a single-question panel is 8 lines), not the nominal line count. If that separator bug is
ever fixed, `RESERVED_CHROME_ROWS` and `MIN_TRACE_ROWS` need recalibration; both are named and documented
for that reason.

- T1 + T2 implemented test-first on `feat/ask-panel-minimize` (not committed).
  RED captured with a signature stub for T1 and the unimplemented options for T2;
  GREEN after implementation. Observed commands/results:
  - `node --experimental-strip-types --test tests/ask-panel-policy.test.ts tests/questionnaire-view.test.ts tests/ask-user-question.test.ts tests/questionnaire-schema.test.ts`
    → 73 tests, 73 pass, 0 fail.
  - `pnpm run typecheck` → `types: 187 recorded diagnostic(s), no regressions`.
- Files: new `lib/ask-panel-policy.ts`, new `tests/ask-panel-policy.test.ts`;
  edited `lib/questionnaire/questionnaire-view.ts`, `tests/questionnaire-view.test.ts`,
  `extensions/ask-user-question.ts`.
- T3-T5 remain for later tasks.

T3 + T4 docs (bounded writer, branch `feat/ask-panel-minimize`, base `main` = `4fcddc2f`): implemented test-first.

- T3: `Ask` added to `CustomizeCategory`; `/gentle:customize` registers two rows (`Ask panel: default state` and
  `Ask panel: minimized indicator`) built with the existing `add` helper and writing through
  `writeAskPanelPreferences` with the command's existing config home. Each row preserves the other
  preference, notifies `Ask panel saved. Applies to the next questionnaire.` and updates in-memory label
  state. RED captured at assertion level (both new Ask tests failed on the missing category, 310 pass / 2 fail)
  before implementation; GREEN after — `node --experimental-strip-types --test tests/gentle-shell.test.ts
  tests/visual-customize-view.test.ts tests/ask-panel-policy.test.ts tests/questionnaire-view.test.ts`
  → 312 tests, 312 pass, 0 fail. The Cards-rows test helper `findCustomizeRow` category bound was raised from
  10 to 14 because `Ask` shifts `Reset` past the old bound.
- T4 docs only: `docs/gentle-shell.md` — the `ask_user_question` bullet now mentions the minimized status bar,
  and a new `### Minimizing the ask panel` subsection documents the toggle key and bar click, the three indicator
  variants, the three default states (`expanded` default), minimized input semantics, the dock-swap scrolling, and
  the `ask-panel.json` persistence through `/gentle:customize` → **Ask**.
- `pnpm run typecheck`: expected `types: N recorded diagnostic(s), no regressions` (see run below).
- T4's full verification set (`pnpm test`, `pnpm run check:runtime-modules`, `node scripts/verify-package-files.mjs`)
  is left to the parent, which runs the complete gate after this handoff.
- Pre-existing pi-lens findings in `lib/visual-customize-view.ts` (nested ternaries) and `extensions/gentle-shell.ts`
  (`as unknown as` without a SAFETY comment) are untouched; neither line is part of this change.

Full verification set (parent, same worktree, after the parent's own review edits):

- `pnpm test`: 4593 tests, 4550 pass, 34 skipped, 9 fail.
- The 9 failures are pre-existing and environmental, proven against the base checkout (`local` =
  `c17f1320`, without this change): two are `tests/yolo-mode-runtime.test.ts` asserting `YOLO: OFF`
  while the shared environment reports `YOLO: ON · session only` (the same file fails 2 of 4 on the
  base, with the identical assertion and render), and the other seven are child-process and timeout
  tests in `tests/agents-session-transport-process.test.ts`, `tests/agents-session-transport.test.ts`,
  `tests/asset-installation-runtime.test.ts`, `tests/review-host-relay-restart-parity.test.ts` and
  `tests/review-host-relay.test.ts`. Running those five files on the base fails 13 unique tests, and
  the set "fails only with this change" is empty.
- `pnpm run check:runtime-modules`: runtime matches TypeScript sources (8 generated modules).
- `node scripts/verify-package-files.mjs`: 155 files, 69 exact byte-pinned contract artifacts.
- `pnpm run typecheck`: 187 recorded diagnostics, no regressions.
- Parent review edits after the writer handoff: the Cards-rows test helper bound became a named
  `CUSTOMIZE_CATEGORY_SCAN_LIMIT` constant with its invariant documented, and the minimized-input
  sentence in `docs/gentle-shell.md` now also names `tab` among the ignored keys.
