# Unit 3 Phase III — Build: Prior Contribution Report

> **⚠️ Timeline disclosure:** This work was completed **2026-04-01 through 2026-04-25**, prior to
> my transfer into the current course unit. It is reported here for Phase III Build documentation
> purposes. All dates, commit hashes, and PR links are verifiable from the repository.

---

## Project

**Repository:** [ioflux-org/studio-json-schema](https://github.com/ioflux-org/studio-json-schema)
**Description:** A visual, interactive, graph-based tool to explore, debug, and understand complex JSON Schemas.
**My fork:** [rattnak/studio-json-schema](https://github.com/rattnak/studio-json-schema)

---

## Implementation Notes

### What I built

Implemented a responsive mobile editor drawer for the JSON Schema Studio. On desktop, the app
displays the Monaco editor and the graph view side-by-side in a horizontal `PanelGroup`. On mobile
(<768px), this layout is unusable — both panels compete for horizontal space, making neither
readable. This PR replaces that layout with a vertical drawer pattern: the graph view takes full
screen width by default, and the Monaco editor slides up from the bottom when the user taps the
toggle button.

Specific features delivered:

1. **Responsive vertical panel layout** — detects mobile breakpoint via `window.innerWidth < 768`
   and a `resize` event listener; switches `PanelGroup` orientation from `horizontal` to `vertical`
   on mobile with the visualization on top and editor on bottom.
2. **Mobile toggle button with vertical chevrons** — `EditorToggleButton` gains an `isMobile?`
   boolean prop; on mobile renders `BsChevronUp`/`BsChevronDown` (vertical) instead of horizontal
   chevrons; 48px touch target meeting WCAG 2.5.5, Apple HIG 44pt, and Android Material 48dp.
3. **Full-height expand overlay** — when the editor is collapsed on mobile, a
   `pointer-events-none` overlay div with a centered `pointer-events-auto` `EditorToggleButton`
   sits at the bottom of the visualization panel so users can re-open the editor.
4. **Virtual keyboard detection via `visualViewport` API** — listens for `visualViewport.resize`
   events; detects keyboard open when `window.innerHeight - viewport.height > 100px`; auto-expands
   the editor panel and clamps resize to `Math.max(40, Math.min(visiblePercent * 0.55, 70))` so
   the editor never shrinks below 40% or exceeds 70% regardless of keyboard height.
5. **Layout viewport fixes** — replaced `h-screen` with `100dvh` in `App.tsx` to account for
   mobile browser dynamic toolbars (address bar show/hide); added `viewport-fit=cover` to
   `index.html` for iOS safe area inset support.
6. **Accessibility** — validation status bar gets `role="status"` and `aria-live="polite"` for
   screen reader announcements; format select gets a visually hidden `<label>` linked via
   `htmlFor`; `EditorToggleButton` gets `aria-expanded` and `aria-label` reflecting open/closed
   state.
7. **Bug fix: validation bar height** — `flex-1` on the validation status bar was consuming excess
   editor height; replaced with `shrink-0`.

### Issue filed

Issue [#260 — Feat: Bottom drawer layout for Monaco editor on mobile screens](https://github.com/ioflux-org/studio-json-schema/issues/260)
was filed by me (`rattnak`) on **2026-04-02**, proposing the vertical drawer pattern. The issue
included a screenshot and alternatives considered (tab-based layout, collapsible panel). I
immediately checked the box indicating willingness to implement it.

### Files modified

| File | What changed |
|------|-------------|
| `src/components/MonacoEditor.tsx` | +159 / -55. Added `isMobile` state + `resize` listener; `visualViewport` keyboard detection with clamped resize math; switched `PanelGroup` orientation on mobile; added `pointer-events-none` expand overlay; `role="status"` + `aria-live` on validation bar; visually hidden `<label>` on format select. |
| `src/components/EditorToggleButton.tsx` | +42 / -13. Added `isMobile?: boolean` prop; conditional vertical chevrons (`BsChevronUp`/`BsChevronDown`); 48px touch target; `aria-expanded` and `aria-label`. |
| `src/App.tsx` | +5 / -3. Replaced `h-screen` with `100dvh`; added `flex flex-col overflow-hidden` to root container. |
| `.changeset/heavy-chicken-relax.md` | Added minor changeset entry documenting the mobile layout change. |

---

## Code Changes

- **Branch:** `rattnak:feat/mobile-editor-drawer-260`
- **Key commits:**

| Hash | Date | Message |
|------|------|---------|
| `efc923a` | 2026-04-02 | `fix: add viewport-fit=cover for iOS safe area support` |
| `87f3145` | 2026-04-02 | `fix: replace h-screen with 100dvh to support mobile browser dynamic toolbar` |
| `d8bdaa1` | 2026-04-02 | `feat: add mobile toggle button with vertical chevrons and 48px touch target for WCAG and Android compliance` |
| `57c6dad` | 2026-04-02 | `feat: add mobile editor drawer with vertical layout, accessible validation bar, and labeled format select` |
| `45eee34` | 2026-04-02 | `feat: auto-expand editor panel when virtual keyboard opens on mobile` |
| `f71145c` | 2026-04-02 | `fix: clamp editor resize to never shrink below 40% when virtual keyboard opens per coderabbit` |
| `867d153` | 2026-04-02 | `fix: prevent expand button overlay from blocking pointer events on SchemaVisualization per coderabbit` |
| `ce3ce19` | 2026-04-05 | `fix: add ARIA semantics and accessible label to error toast per coderabbit suggestion` |
| `c9b96a2` | 2026-04-25 | Merge commit — `feat: mobile editor drawer layout improvements (#263)` |

- **Linked issue:** [#260 — Feat: Bottom drawer layout for Monaco editor on mobile screens](https://github.com/ioflux-org/studio-json-schema/issues/260)
  - Filed by: `rattnak` on 2026-04-02
- **PR:** [#263 — feat: mobile editor drawer layout improvements](https://github.com/ioflux-org/studio-json-schema/pull/263)
  - Opened: 2026-04-02
  - Merged: **2026-04-25** by `AgniveshChaubey` (maintainer)
  - Status: **MERGED ✅**

---

## Testing Strategy

### Tests added

None merged into the repo. The project has no automated test suite — confirmed by maintainer in
prior PR #157 review: *"Currently we don't have any configuration for automated testing."* The
CodeRabbit pre-merge check flagged **0% docstring coverage** (threshold 80%) as a warning; it did
not block the merge.

### CI validation on PR #263

| Workflow | Result | What it checked |
|----------|--------|----------------|
| `build-preview.yml` | ✅ Passed | `npm ci` + `npm run build` (TypeScript compile + Vite bundle) |
| `check-changeset.yml` | ✅ Passed | `.changeset/heavy-chicken-relax.md` present |
| `deploy-preview.yml` | ✅ Deployed | Cloudflare Pages preview at `pr-263.studio-json-schema.pages.dev` |
| CodeRabbit pre-merge checks | ⚠️ 1 warning | Docstring coverage 0% (warning only, not blocking) |

No lint or type-check CI step is present in any workflow file.

### Maintainer review feedback addressed

| Date | Who | Feedback | My response |
|------|-----|----------|-------------|
| 2026-04-02 | `coderabbitai` (inline) | Keyboard-open resize math shrank the editor on tall keyboards: `(viewport.height / window.innerHeight) * 0.55` produced values below 40% on 800px screen + 300px keyboard | Fixed in commit `f71145c`: rewrote to `Math.max(40, Math.min(Math.round(visiblePercent * 0.55), 70))` with correct `visiblePercent` derivation |
| 2026-04-02 | `coderabbitai` (inline) | Full-width `z-10` wrapper around expand overlay CTA blocked all pointer events to `SchemaVisualization` on the empty bottom strip | Fixed in commit `867d153`: added `pointer-events-none` to wrapper, `pointer-events-auto` to button only |
| 2026-04-02 | `coderabbitai` (nitpick) | `EditorToggleButton` missing `aria-expanded` to expose drawer state to assistive tech | Addressed: `aria-expanded` and `aria-label` added |
| 2026-04-05 | Follow-up | ARIA semantics for error toast | Fixed in commit `ce3ce19` |
| 2026-04-25 | `AgniveshChaubey` | Final approval | APPROVED — merged |

---

## Challenges Faced

All items below are sourced from the verifiable PR/issue comment thread via the GitHub API.

### 1. Incorrect `visualViewport` resize math — editor shrank on tall keyboards

The initial implementation computed the editor resize percentage as
`(viewport.height / window.innerHeight) * 100 * 0.55`. CodeRabbit flagged this on 2026-04-02 with
a concrete counter-example: on an 800px screen with a 300px keyboard, `viewport.height = 500`,
so the formula yielded `(500/800) * 100 * 0.55 ≈ 34%` — *smaller* than the 40% mobile default,
pushing the editor further off-screen when the keyboard opened instead of expanding it.

The fix required rethinking the math: instead of scaling from `viewport.height`, the correct
approach is to compute how much of the total height the keyboard consumes, then determine what
fraction of *visible* space the editor should occupy, and clamp the result:
`Math.max(40, Math.min(Math.round(visiblePercent * 0.55), 70))`. This was fixed in `f71145c`.

### 2. Expand button overlay blocking graph interaction

The `pointer-events` issue was a non-obvious CSS layering bug: the collapse CTA wrapper was a
full-width `z-10` div spanning the entire bottom strip. Even though the button itself was small,
the invisible padding area of the wrapper intercepted all touch/click events destined for the graph
below it. CodeRabbit caught this. The fix (`867d153`) was `pointer-events-none` on the wrapper and
`pointer-events-auto` scoped to the button only.

### 3. Scope creep triggered maintainer feedback (on PR #245, initially attributed to #263)

On 2026-04-02 — the same day the PR opened — maintainer `AgniveshChaubey` posted a detailed comment
expressing concern that the PR felt AI-generated and exceeded its scope, with many unrelated changes
bundled in. He spent close to an hour reviewing it and stated it would have been faster to implement
the feature himself. He removed the out-of-scope changes.

This comment was later clarified (2026-04-05) to have been intended for PR #245 (unified search
bar), not #263. However, the experience informed how subsequent PRs were scoped and communicated.
The response (2026-04-02) acknowledged the scope creep, explained that AI was used for WCAG/UI
standards checking (not code generation), and committed to keeping PRs minimal and flagging scope
expansion earlier.

### 4. Extended review cycle — 23 days from open to merge

PR opened 2026-04-02, merged 2026-04-25 — 23 days. Contributing factors:
- Waiting for `AgniveshChaubey` review after CodeRabbit issues were resolved (follow-up requested
  2026-04-07; maintainer confirmed review by weekend 2026-04-07)
- Two merge-from-main syncs required during the wait (`77e5e0e` on 2026-04-25) to resolve conflicts
- Final APPROVED by `AgniveshChaubey` on 2026-04-25T18:47Z, merged same day

### 5. Self-filed issue — had to justify the feature scope upfront

Issue #260 was filed by me before the PR, which required writing a full problem statement,
proposed solution, screenshots, and alternatives considered to establish legitimacy of the feature
request. This was a direct lesson learned from the PR #157 experience where a PR was opened without
an assigned issue.

---

## Link to Merged PR

[https://github.com/ioflux-org/studio-json-schema/pull/263](https://github.com/ioflux-org/studio-json-schema/pull/263)
