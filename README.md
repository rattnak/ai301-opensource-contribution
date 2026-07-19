# Open Source Contribution Report

This report tracks two contributions. Contribution 2 (Open Library) is in progress; Contribution 1 (ioflux) is complete and merged. Each follows the standard contribution template.

---

# Contribution 2: Trusted Book Providers Import Process (Open Library)

* **Contribution Number:** 2
* **Student:** Chanrattnak Mong
* **Issue:** No dedicated GitHub issue opened yet — targeting [#10856](https://github.com/internetarchive/openlibrary/issues/10856) (BookDash import, open/assigned) as the concrete scope. Background/prior art: [#8462](https://github.com/internetarchive/openlibrary/issues/8462), [#12091](https://github.com/internetarchive/openlibrary/issues/12091), [#8542](https://github.com/internetarchive/openlibrary/issues/8542)
**Status:** Phase I → II — Recommendation posted on #10856, awaiting assignment/maintainer confirmation

---

## Note on Issue Status

There is no dedicated GitHub issue opened for this contribution yet. A maintainer informed me directly (see screenshot below) to research the Trusted Book Providers (TBP) import process and come back with a concrete recommendation.

After reading through the codebase (`docs/ai/imports/`), the four related issues, and comparing the two candidate data sources in flight (ITAN #12091 and BookDash #10856), I've landed on a concrete recommendation: **pick up the stalled BookDash import (#10856)** rather than starting a brand-new source from scratch. Reasoning:

- ITAN's adapter code lives in a separate repo (`openlibrary-bots`) and is blocked on an identifier-registration PR merging first — higher setup cost, less in my control.
- BookDash already has a working scraper (contributor's gist) and a real batch already submitted (`openlibrary.org/import/batch/1516`), stuck on one specific, well-defined bug: the scraper appended author roles to names (e.g. `"Megan Andrews (Illustrator)"`), risking duplicate author records on import. The issue is still open and assigned, but no fix has landed in months.
- This is a smaller, closeable task that produces a real worked example for the "streamline & document" goal, rather than a second half-finished source.

I posted this recommendation directly as a comment on [issue #10856](https://github.com/internetarchive/openlibrary/issues/10856) and asked the maintainer to assign it to me. Once assigned/confirmed, I'll update this section with the comment link and, if a dedicated issue is opened, that link as well.

**Screenshot of maintainer conversation:**
<img width="888" height="456" alt="Screenshot 2026-07-14 at 8 20 20 PM" src="https://github.com/user-attachments/assets/7bccadff-b37a-4c17-891a-7840a40752e9" />


---

## Why I Chose This Issue

I wanted a contribution that involved understanding an existing data pipeline end-to-end rather than a single isolated UI fix, and Open Library's Trusted Book Providers (TBP) program fit that well — it touches data modeling, external partner integration, and a real backlog of stalled community requests that a clearer process could unblock. This specific direction came from a maintainer who asked me to research the TBP import process directly, rather than from picking an issue off a list, which also means the eventual dedicated issue will already have maintainer buy-in. It follows the scope-creep lesson from my prior ioflux contribution — I'm confirming scope directly with a maintainer through research before writing any code.

---

## Understanding the Issue

### Problem Description

Onboarding a new "trusted" external book data source (e.g. a publisher's catalog) into Open Library currently requires a maintainer to manually walk each contributor through writing a custom Python script, extending TBP registration code, and submitting records via `openlibrary-client` — a process that isn't well documented outside git history and ad hoc conversations. The related issues (#8462, #8542) point at this same documentation/tooling gap; the maintainer asked me to research it directly.

### Expected Behavior

A new trusted provider should be onboardable by following documented steps and a reference adapter implementation, with a maintainer needed only for final identifier registration and batch approval — not for explaining the pipeline from scratch each time.

### Current Behavior

Each new source is a bespoke, mostly undocumented effort. Two related public issues show this breaking down in different ways: ITAN (#12091) is blocked on an identifier-registration PR that hasn't merged/deployed; BookDash (#10856) got further (working scraper, one real batch submitted as `openlibrary.org/import/batch/1516`) but is stuck on an undocumented gotcha — the scraper appended author roles to names (e.g. `"Megan Andrews (Illustrator)"`), which risks creating duplicate author records instead of matching the existing canonical author. A maintainer said months ago they'd try to fix this on the OL side; nothing has landed, and the batch remains unresolved with no UI path to edit or delete it.

### Affected Components

- `openlibrary/plugins/importapi/` — HTTP import endpoints (`/api/import`, `/api/import/ia`)
- `openlibrary/core/batch_imports.py` and `openlibrary/core/imports.py` — batch JSONL ingestion and the `Batch` queue model
- `openlibrary/catalog/add_book/__init__.py` — final record creation, dedup matching, and the hardcoded `ALLOWED_COVER_HOSTS` allowlist
- `openlibrary/plugins/openlibrary/config/edition/identifiers.yml` — identifier type registration
- `openlibrary-client` (separate repo) — `olclient/imports.py`, `import.schema.json`
- `openlibrary-bots` (separate repo) — `sources/<slug>/` concrete source adapters

---

## Reproduction Process

### Environment Setup

Not started — no local development environment set up yet. Per program guidelines, environment setup and fork cloning happen in Phase II, once a dedicated issue exists.

### Steps to Reproduce

Not applicable yet — this is a process/documentation gap, not a reproducible bug. Phase II will involve tracing the pipeline hands-on (likely via the documented local setup in `docs/ai/imports/debugging.md`) once scope is confirmed with the maintainer.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — Phase I, no code yet
- **Screenshots/logs:** N/A
- **My findings:** N/A — reserved for Phase II

---

## Solution Approach

### Analysis

The root cause is a name-normalization gap: whatever validates/ingests batch JSONL records doesn't strip or flag parenthetical role suffixes on author names before they're matched against existing author records in `add_book.load()`. The fix could live in one of two places — the source-specific scraper (quick, but only fixes BookDash) or the shared import pipeline's author-matching step in `openlibrary/catalog/add_book/__init__.py` (slower, but prevents every future source from hitting the same bug). The maintainer's own comment on #10856 ("I think we'll try to update these for you") suggests they lean toward a pipeline-side fix, but this needs to be confirmed before scoping the PR.

### Proposed Solution

1. Pull the exact author-string format from the contributor's scraper gist to confirm the pattern to strip (e.g. trailing `" (Role)"`).
2. Confirm with the maintainer whether the fix belongs in the shared pipeline or the one-off script.
3. Implement the normalization + a regression test.
4. Dry-run corrected records via `/api/import?preview=true` (or the FastAPI `/import/preview.json`) before touching the real batch.
5. Coordinate deleting/resubmitting batch 1516 (no in-UI edit/delete exists today).
6. Document the "parenthetical author role" gotcha in `docs/ai/imports/adding-sources.md`, and separately flag that the doc links to a `pm/workflows/import_workflow.md` file that doesn't exist in the repo.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Traced the full pipeline via `docs/ai/imports/` and `openlibrary/catalog/add_book/__init__.py` to find where author matching happens and where a name-normalization fix would need to live.

**Match:** Compared fixing the one-off scraper vs. a shared pipeline-side normalization step (the latter reuses the existing author-matching code path in `add_book.load()` rather than adding source-specific logic that only benefits BookDash).

**Plan:** Documented as a 6-step plan above — confirm fix location with maintainer, implement + test, dry-run via preview endpoint, resubmit batch 1516, update docs, flag the broken doc link. See working plan notes for full detail.

**Implement:** Not yet started — pending maintainer confirmation on pipeline-side vs. script-side fix.

**Review:** Not yet started — Phase III.

**Evaluate:** Not yet started — Phase III.

---

## Testing Strategy

### Unit Tests

- [ ] [Not yet started — Phase III]

### Integration Tests

- [ ] [Not yet started — Phase III]

### Manual Testing

[Not yet started — Phase III]

---

## Implementation Notes

### Week 1 Progress (2026-07-07)

Spent this week on research only, no code written. Read the four related GitHub issues (#8462, #10856, #12091, #8542) in full, including comment threads, per a maintainer's direct request to investigate the TBP import process. Compared the ITAN and BookDash cases as reference points for what a streamlined process needs to solve.

### Week 2 Progress (2026-07-14)

Reviewed `docs/ai/imports/` (recently merged internal documentation covering the pipeline architecture, API endpoints, and how to add a new source) against the four issues to see what's already solved vs. still a real gap. Decided on a concrete recommendation: pick up the stalled BookDash batch (#10856) rather than starting a new source, since it's the smallest closeable unit of work with a real, already-submitted batch blocked on one specific bug (author name/role duplication risk). Confirmed the issue is still open and assigned (no fix landed) and drafted a 6-step implementation plan (see Solution Approach above). Also found that `docs/ai/imports/adding-sources.md` links to a `pm/workflows/import_workflow.md` file that doesn't exist anywhere in the repo — a small doc gap worth flagging to the maintainer alongside the main recommendation.

### Week 3 Progress (2026-07-19)

Posted the recommendation and 6-step plan as a comment directly on [issue #10856](https://github.com/internetarchive/openlibrary/issues/10856), and asked the maintainer to assign the issue to me. Also sent a short direct follow-up message since I'd been slow to respond earlier in the week due to other work and coursework commitments. Awaiting maintainer response on: (1) whether to fix the author-name bug in the scraper script or the shared pipeline, (2) how to handle deleting/resubmitting the stuck batch, and (3) confirmation to proceed with BookDash as the scoped source.

### Code Changes

- **Files modified:** None yet
- **Key commits:** None yet
- **Approach decisions:** Targeting BookDash (#10856) over ITAN (#12091) as the scoped source — see Solution Approach for reasoning. Pipeline-side vs. script-side fix location still pending maintainer confirmation.

---

## Pull Request

**PR Link:** Not yet opened

**PR Description:** [Not yet started]

**Maintainer Feedback:**
- 2026-07-14: Maintainer asked me directly (via chat) to research the TBP import process and come back with a recommendation, rather than assigning a specific existing issue.
- 2026-07-19: Posted the recommendation on issue #10856 and requested assignment; awaiting maintainer response.

**Status:** Awaiting maintainer response / assignment on #10856

---

## Learnings & Reflections

### Technical Skills Gained

Early-stage: learned the shape of a real production import/ETL pipeline from documentation and issue threads — schema validation, deduplication by identifier, a human-in-the-loop batch review queue, and a few silent-failure modes (identifier drop, cover-host drop) worth designing around later.

### Challenges Overcome

Working without a formally assigned issue required extra care to keep the "Why I Chose This Issue" and issue-tracking sections honest about what's confirmed vs. still pending, rather than presenting related issues as if they were the actual assignment.

### What I'd Do Differently Next Time

Ask for a dedicated issue to be filed at the start of the conversation, rather than after research is complete, so tracking stays unambiguous from day one.

---

## Resources Used

- [docs/ai/imports/](https://github.com/internetarchive/openlibrary/tree/master/docs/ai/imports) (internal Open Library documentation)
- [Issue #8462](https://github.com/internetarchive/openlibrary/issues/8462), [#12091](https://github.com/internetarchive/openlibrary/issues/12091), [#10856](https://github.com/internetarchive/openlibrary/issues/10856), [#8542](https://github.com/internetarchive/openlibrary/issues/8542)
- [BookDash scraper gist](https://gist.github.com/sai-krishna-kotha/2602c9d13b0f3da4c4a0e701096b0219)
- [openlibrary-client import schema](https://github.com/internetarchive/openlibrary-client/blob/master/olclient/schemata/import.schema.json)

---

# Contribution 1: Responsive Mobile Editor Drawer for JSON Schema Studio

**Contribution Number:** 1
**Student:** Rattnak
**Issue:** [ioflux-org/studio-json-schema#260 — Feat: Bottom drawer layout for Monaco editor on mobile screens](https://github.com/ioflux-org/studio-json-schema/issues/260)
**Status:** Phase IV — Complete

> **⚠️ Timeline disclosure:** The work below was completed **2026-04-01 through 2026-04-25**, prior to transfer into the current course unit. It is reported here for Phase III Build and Phase IV Submit & Iterate documentation purposes. All dates, commit hashes, and PR links are verifiable from the repository.

**Repository:** [ioflux-org/studio-json-schema](https://github.com/ioflux-org/studio-json-schema)
**Description:** A visual, interactive, graph-based tool to explore, debug, and understand complex JSON Schemas.
**My fork:** [rattnak/studio-json-schema](https://github.com/rattnak/studio-json-schema)

---

## Why I Chose This Issue

I'd already contributed once to this project (PR #157) and hit feedback that a PR without a linked issue lacked context for the maintainer to evaluate scope. This next contribution was a chance to apply that lesson directly: file the issue first, get the problem and proposed solution on record, then implement. The mobile layout problem was also something I could evaluate hands-on — the desktop side-by-side panel layout was visibly unusable on a phone-width viewport, a concrete, scoped UI problem well suited to practicing accessibility-aware frontend work (WCAG touch targets, ARIA semantics, viewport handling).

---

## Understanding the Issue

### Problem Description

On screens narrower than 768px, the Monaco editor and graph view were forced into the same horizontal `PanelGroup` split used on desktop, leaving both panels too narrow to be usable.

### Expected Behavior

On mobile, the graph view should take full screen width by default, with the editor accessible as a bottom drawer the user can expand/collapse — including graceful behavior when the on-screen keyboard opens.

### Current Behavior

Both panels competed for horizontal space at mobile widths, making neither the code editor nor the graph readable or usable.

### Affected Components

- `src/components/MonacoEditor.tsx` — panel orientation, keyboard-aware resize logic, accessibility attributes
- `src/components/EditorToggleButton.tsx` — mobile toggle button and touch target sizing
- `src/App.tsx` — viewport height handling (`100dvh`, safe-area insets)

---

## Reproduction Process

### Environment Setup

Standard local clone + `npm` install/dev server; no unusual setup issues.

### Steps to Reproduce

1. Open the app in a browser at a viewport width under 768px.
2. Observe the Monaco editor and graph view rendered side-by-side, both too narrow to use.
3. Observed result: neither panel is legible or usable at mobile widths.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — captured via screenshot in issue #260
- **Screenshots/logs:** Included in issue [#260](https://github.com/ioflux-org/studio-json-schema/issues/260)
- **My findings:** The existing `PanelGroup` component only supported a horizontal orientation; a vertical drawer pattern was needed, along with keyboard-aware resize handling since Monaco is a heavy interactive component that behaves poorly when squeezed by the mobile virtual keyboard.

---

## Solution Approach

### Analysis

The root cause was that the layout was built desktop-first with no responsive breakpoint logic, and `PanelGroup` orientation was hardcoded to horizontal.

### Proposed Solution

Detect mobile breakpoint (`window.innerWidth < 768`) and switch `PanelGroup` to vertical orientation, with the graph on top and the editor as a bottom drawer that slides open via a toggle button, plus `visualViewport`-based keyboard detection to resize the editor safely when the keyboard opens.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Mobile users can't use the app because two panels can't coexist at narrow widths.

**Match:** Looked at how the existing `EditorToggleButton` and `PanelGroup` were structured to extend rather than replace them; considered a tab-based layout and a collapsible panel as alternatives (documented in issue #260) before settling on the drawer pattern.

**Plan:**
1. Add mobile detection + `resize` listener; switch `PanelGroup` orientation on mobile.
2. Add `isMobile` prop and vertical chevrons to `EditorToggleButton`, sized to meet WCAG/HIG/Material touch-target minimums.
3. Add a `pointer-events-none` expand overlay so the collapsed-state CTA doesn't block the graph.
4. Add `visualViewport`-based keyboard detection with clamped auto-resize.
5. Fix viewport height (`100dvh`) and iOS safe-area handling.
6. Add ARIA semantics to the validation bar and format select.

**Implement:** Branch `rattnak:feat/mobile-editor-drawer-260`; commits listed under Code Changes below.

**Review:** Self-reviewed for scope against issue #260 before opening the PR, given the PR #157 feedback about missing linked issues.

**Evaluate:** Manual testing across mobile viewport widths and keyboard-open states; CI build/deploy-preview checks; CodeRabbit automated review.

---

## Testing Strategy

### Unit Tests

- [x] N/A — project has no automated test suite (confirmed by maintainer in PR #157 review: *"Currently we don't have any configuration for automated testing."*)

### Integration Tests

- [x] N/A — no integration test harness present in the repo

### Manual Testing

Manually tested the vertical drawer layout, toggle button expand/collapse, virtual-keyboard-open resize behavior, and screen-reader announcements (`role="status"`, `aria-live`) across mobile viewport widths via the Cloudflare Pages preview deploy (`pr-263.studio-json-schema.pages.dev`).

---

## Implementation Notes

### Week of 2026-04-02 Progress

Built the full feature in a single day: responsive panel orientation, mobile toggle button with proper touch targets, keyboard-aware resize clamping, viewport fixes, and accessibility attributes. CodeRabbit caught two real functional bugs same-day (keyboard-resize math, pointer-events overlay blocking), both fixed same day. A maintainer comment raising AI-generated/scope-creep concerns arrived the same day; later clarified (2026-04-05) to have been intended for a different PR (#245), but it shaped how I communicated AI-assistance boundaries going forward.

### Week of 2026-04-25 Progress

After a quiet review period, followed up once (2026-04-07) rather than repeatedly pinging. Two merge-from-main syncs were needed to resolve conflicts. Final maintainer approval and merge landed 2026-04-25.

### Code Changes

- **Files modified:**
  - `src/components/MonacoEditor.tsx` (+159/-55): mobile state + resize listener; `visualViewport` keyboard detection with clamped resize math; `PanelGroup` orientation switch; `pointer-events-none` expand overlay; `role="status"`/`aria-live` on validation bar; hidden `<label>` on format select
  - `src/components/EditorToggleButton.tsx` (+42/-13): `isMobile?` prop; conditional vertical chevrons; 48px touch target; `aria-expanded`/`aria-label`
  - `src/App.tsx` (+5/-3): `100dvh` instead of `h-screen`; `flex flex-col overflow-hidden` on root container
  - `.changeset/heavy-chicken-relax.md`: changeset entry
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

- **Approach decisions:** Chose the vertical drawer pattern over a tab-based layout or simple collapsible panel (alternatives documented in issue #260) because it preserves full-width graph visibility by default, which is the primary mobile use case.

---

## Pull Request

**PR Link:** [ioflux-org/studio-json-schema#263](https://github.com/ioflux-org/studio-json-schema/pull/263)

**PR Description:** Implemented a responsive mobile drawer layout for the JSON Schema Studio editor: on screens narrower than 768px, the horizontal Monaco-editor/graph-view split is replaced with a vertical drawer where the graph takes full width and the editor slides up from the bottom. Adds virtual-keyboard-aware resize clamping, a 48px accessible toggle button with proper ARIA attributes, and viewport fixes (`100dvh`, `viewport-fit=cover`) for mobile browser chrome and iOS safe areas. Closes issue #260, which I filed myself before starting implementation.

**Maintainer Feedback:**
- 2026-04-02 (`coderabbitai`, inline): keyboard-resize math could shrink the editor below the 40% floor on tall keyboards → fixed in `f71145c`
- 2026-04-02 (`coderabbitai`, inline): full-width `z-10` overlay wrapper blocked pointer events to the graph → fixed in `867d153`
- 2026-04-02 (`coderabbitai`, nitpick): toggle button missing `aria-expanded` → addressed
- 2026-04-05 (follow-up review): ARIA semantics missing on error toast → fixed in `ce3ce19`
- 2026-04-02 (`AgniveshChaubey`, maintainer): concern about AI-generated feel / scope creep, later clarified to have been intended for a different PR (#245) → acknowledged, explained AI was used only for WCAG/UI-standards checking, committed to smaller scoped PRs going forward
- 2026-04-25 (`AgniveshChaubey`, maintainer): final review → **APPROVED**, merged same day

**Status:** Approved / Merged ✅ (23 days from open to merge, including two rebases against `main`)

---

## Learnings & Reflections

### Technical Skills Gained

Practiced viewport-relative CSS units (`dvh`) and iOS safe-area handling, the `visualViewport` API for keyboard-aware layout, and WCAG/HIG/Material touch-target sizing — plus treating a `pointer-events` CSS layering bug as a real functional defect, not a style nit.

### Challenges Overcome

- Fixed a genuine math bug in the keyboard-open resize formula (CodeRabbit caught that `(viewport.height / window.innerHeight) * 0.55` could yield a value smaller than the mobile default floor, shrinking the editor exactly when it should expand).
- Fixed an invisible full-width overlay wrapper silently blocking clicks to the graph view underneath it.
- Navigated a maintainer scope-creep comment that was initially (and, for a few days, unresolved-ly) attributed to this PR before being clarified as meant for a different one.

### What I'd Do Differently Next Time

Proactively state the AI-assistance boundary (used for accessibility-standards lookup, not code generation) directly in the PR description, rather than only explaining it after a maintainer raised concern.

---

## Resources Used

- [Issue #260](https://github.com/ioflux-org/studio-json-schema/issues/260)
- [PR #263](https://github.com/ioflux-org/studio-json-schema/pull/263)
- Prior PR #157 review feedback (informed the "file the issue first" approach)
- WCAG 2.5.5, Apple HIG, and Android Material touch-target size guidelines
