# Open Source Contribution Report

This report tracks two contributions. Contribution 2 (Open Library) is in progress; Contribution 1 (ioflux) is complete and merged. Each follows the standard contribution template.

---

# Contribution 2: Streamlining the Trusted Book Providers Import Process

**Contribution Number:** 2
**Student:** Rattnak
**Issue:** [#8462 — Video Tutorial of Implementing a Trusted Book Provider](https://github.com/internetarchive/openlibrary/issues/8462)
**Status:** Phase I — In Progress

---

## Why I Chose This Issue

I wanted a contribution that involved understanding an existing data pipeline end-to-end rather than a single isolated UI fix, and Open Library's Trusted Book Providers (TBP) program fit that well — it touches data modeling, external partner integration, and a real backlog of stalled community requests (ITAN, BookDash) that a clearer process could unblock. Issue #8462 is explicitly a documentation/tooling gap: maintainer `mekarpeles` is asking for a worked example so future partners don't each need hours of maintainer hand-holding. That's a scope I can complete without needing deep pre-existing familiarity with Open Library's Infobase internals, while still learning a nontrivial ingestion pipeline (schema validation, dedup, batch queues, ImportBot).

It also directly follows the scope-creep lesson from my prior ioflux contribution: rather than jumping into code, I'm spending Phase I fully understanding the pipeline and proposing a plan to a maintainer before writing anything, to keep the eventual PR narrowly scoped.

---

## Understanding the Issue

### Problem Description

Onboarding a new "trusted" external book data source (e.g. a publisher's catalog) into Open Library currently requires a maintainer to manually walk each contributor through writing a custom Python script, extending TBP registration code, and submitting records via `openlibrary-client` — a process that is undocumented outside of git commit history and Slack conversations. Issue #8462 asks for a video/worked example; the related issue #8542 flags the same doc gap for batch imports specifically.

### Expected Behavior

A new trusted provider (with a catalog in a reasonably clean format) should be onboardable by a contributor following documented steps and a reference adapter implementation, with a maintainer only needed for final identifier-registration and batch approval — not for explaining the pipeline from scratch each time.

### Current Behavior

Each new source is a bespoke, mostly undocumented effort. Two live examples show the pattern breaking down in different ways:
- **ITAN** ([#12091](https://github.com/internetarchive/openlibrary/issues/12091)): a publisher waited 2+ weeks for a response after submitting their catalog; the request is now blocked on an identifier-registration PR that hasn't merged/deployed.
- **BookDash** ([#10856](https://github.com/internetarchive/openlibrary/issues/10856)): a contributor got much further (working scraper, one real batch submitted) but hit undocumented gotchas — author-name/role duplication risk, silently dropped cover images, no way to delete a bad batch — that a documented pipeline would have surfaced earlier.

### Affected Components

- `openlibrary/plugins/importapi/` — HTTP import endpoints (`/api/import`, `/api/import/ia`)
- `openlibrary/core/batch_imports.py` and `openlibrary/core/imports.py` — batch JSONL ingestion and the `Batch` queue model
- `openlibrary/catalog/add_book/__init__.py` — final record creation, dedup matching, and the hardcoded `ALLOWED_COVER_HOSTS` allowlist
- `openlibrary/plugins/openlibrary/config/edition/identifiers.yml` — identifier type registration
- `openlibrary-client` (separate repo) — `olclient/imports.py` (`DataProvider`, `OLImportRecord`) and `import.schema.json`
- `openlibrary-bots` (separate repo) — `sources/<slug>/` concrete source adapters (e.g. the in-progress ITAN adapter)
- `docs/ai/imports/` — existing internal documentation of the pipeline, which I'm treating as the current source of truth

---

## Reproduction Process

### Environment Setup

Not yet applicable — Phase I has been research against the live GitHub repo and issue threads rather than a local dev environment. The repo's `docs/ai/imports/debugging.md` documents a local setup path (`compose.near-prod.yaml` with an `importbot` service, `LOCAL_DEV=true` seeded dev login) that I plan to use once implementation starts, to run a full submit → queue → ImportBot cycle locally before proposing changes against the real batch queue.

### Steps to Reproduce

This issue isn't a bug with a repro path — it's a process gap. The "reproduction" was tracing the two blocked real-world cases through the pipeline:

1. Read issue #8462 (tracking issue) and #8542 (docs gap) to confirm the maintainer's ask.
2. Read #12091 (ITAN) end-to-end — found the identifier-registration blocker.
3. Read #10856 (BookDash) end-to-end — found the batch already exists (`openlibrary.org/import/batch/1516`) with specific, named defects still open.
4. Cross-referenced both against `openlibrary/plugins/importapi/` source and `docs/ai/imports/` to see exactly where each case stalls in the pipeline.

### Reproduction Evidence

- **Commit showing reproduction:** N/A (research phase, no code changes yet)
- **Screenshots/logs:** N/A
- **My findings:** See "Understanding the Issue" and "Analysis" — the pipeline works, but has three specific undocumented failure points (identifier-registration timing, cover-host allowlist, author-name parsing) that stall real contributors without maintainer intervention.

---

## Solution Approach

### Analysis

The pipeline itself (`DataProvider` → `OLImportRecord` → `/import/batch/new` → `Batch` queue → ImportBot → `add_book.load()`) is functional and already used in production. The root cause of onboarding friction isn't the pipeline's design — it's that its constraints are undocumented and only discoverable by hitting them:
1. Identifier types must be registered in `identifiers.yml` and **deployed** before a batch references them, or the field is silently dropped (no error).
2. Cover URLs are silently set to `None` if the host isn't in a hardcoded allowlist in `add_book/__init__.py` — again, no error.
3. There's no batch-delete capability, so a data-quality mistake (like BookDash's author-role-in-name bug) can't be cleanly retried.

These are exactly the kind of gotchas a documented, worked example would surface up front instead of via a multi-week back-and-forth with a maintainer.

### Proposed Solution

Use BookDash as the concrete worked example, since it's furthest along and already has maintainer engagement:
1. Fix the two known BookDash defects (author-role stripping in the scraper/adapter output, and either expanding the cover-host allowlist or re-hosting BookDash cover images) as a first, narrowly-scoped PR.
2. Formalize the BookDash scraper into the standard `openlibrary-bots/sources/<slug>/` adapter pattern (`DataProviderRecord` + `JSONLProvider`), rather than leaving it as a one-off script.
3. Write up the end-to-end steps as the documentation #8462 and #8542 are asking for, using BookDash as the reference case.

This also unblocks ITAN indirectly: once the identifier-registration-before-deploy constraint and adapter pattern are clearly documented, ITAN's remaining blocker (PR #12947 merging/deploying) becomes a simple sequencing step rather than a source of repeated confusion.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** New trusted-provider onboarding is undocumented and maintainer-bottlenecked; two real requests (ITAN, BookDash) are stalled for different, both-documentable reasons.

**Match:** The existing `docs/ai/imports/` directory already documents architecture, API endpoints, and known limitations in detail — my job is to extend/validate this against a real end-to-end case (BookDash) rather than invent new documentation structure from scratch. The `openlibrary-bots/sources/itan/` adapter (in progress, PR #447) is a template for what a completed BookDash adapter should look like.

**Plan:**
1. Confirm with maintainer (`mekarpeles`/`cdrini`) that BookDash is an acceptable pilot case before starting (avoids the scope-creep issue from Contribution 1).
2. Fix author-role parsing in the BookDash scraper output so authors aren't duplicated on import.
3. Resolve the cover-image gap — propose either an `ALLOWED_COVER_HOSTS` PR for `bookdash.org` or a re-hosting step.
4. Package the scraper as a proper `DataProviderRecord`/`JSONLProvider` adapter under `openlibrary-bots/sources/bookdash/`.
5. Validate with `?preview=true` dry-runs before any real batch resubmission.
6. Write/extend documentation in `docs/ai/imports/adding-sources.md` using BookDash as the worked example.

**Implement:** Not started — pending maintainer sign-off on scope (see Status below).

**Review:** Self-review checklist once implemented: PR touches only the scoped files (scraper fix, adapter, docs); no unrelated refactors; `source_records` prefix stable; dry-run output included in PR description per the project's own PR review expectations (`docs/ai/imports/adding-sources.md`).

**Evaluate:** Local dry-run via `?preview=true`, plus local ImportBot run via `compose.near-prod.yaml` before proposing any real batch changes.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Author name parsing strips role suffixes (e.g. `"Name (Illustrator)"` → `"Name"`) without over-stripping legitimate parenthetical name content
- [ ] Test case 2: Cover URL host-check behavior against the (possibly expanded) `ALLOWED_COVER_HOSTS` list
- [ ] Test case 3: Adapter's `to_ol_import()` correctly returns `None` for malformed source records instead of raising

### Integration Tests

- [ ] Integration scenario 1: End-to-end `?preview=true` dry-run of a BookDash record through the adapter → `OLImportRecord` → validator
- [ ] Integration scenario 2: Local ImportBot queue processing of a small resubmitted BookDash batch (via `compose.near-prod.yaml`)

### Manual Testing

Not yet started — planned once implementation begins, using the local dev pipeline documented in `docs/ai/imports/debugging.md`.

---

## Implementation Notes

### Week 1 Progress (2026-07-14)

Spent this week entirely on research, no code written yet. Read issues #8462, #12091, #10856, and #8542 in full, including comment threads. Found that the repo already has substantial internal documentation at `docs/ai/imports/` (index, API reference, adding-sources guide, debugging/known-limitations doc) that closely maps to what #8462/#8542 are asking for — meaning this contribution is more "close the gap and validate against a real case" than "write from a blank page." Compared ITAN vs. BookDash as pilot candidates and concluded BookDash is the better first case (further along, maintainer already engaged, concrete named defects rather than open design questions). Next step is proposing this plan to a maintainer on Slack before touching code.

### Code Changes

- **Files modified:** None yet
- **Key commits:** None yet
- **Approach decisions:** Chose BookDash over ITAN as the pilot case (see Analysis); deferred any implementation until maintainer scope confirmation, per the lesson learned in Contribution 1

---

## Pull Request

**PR Link:** Not yet opened

**PR Description:** Draft plan (pending maintainer confirmation): fix BookDash author-role and cover-host defects, formalize the BookDash scraper into the standard `openlibrary-bots` adapter pattern, and extend `docs/ai/imports/adding-sources.md` with BookDash as the worked example — addressing #8462 and #8542, and unblocking #10856.

**Maintainer Feedback:**
- Not yet solicited — next step is reaching out on the OpenLibrary Slack to confirm BookDash as an acceptable pilot before implementation.

**Status:** Scoping — pre-PR

---

## Learnings & Reflections

### Technical Skills Gained

Learned the shape of a real production import/ETL pipeline: schema validation with `extra="forbid"` Pydantic models, deduplication by identifier, a human-in-the-loop batch review queue, and the specific silent-failure modes (identifier drop, cover-host drop) that this kind of pipeline accumulates over time without always being one loud error.

### Challenges Overcome

The main challenge so far was disambiguating "what's already solved" from "what's actually the gap" — the repo's existing `docs/ai/imports/` documentation is thorough, so the real contribution isn't writing docs from scratch but finding and closing the specific remaining defects (author parsing, cover allowlist) that stand between "documented pipeline" and "a real partner successfully onboarded."

### What I'd Do Differently Next Time

Get maintainer confirmation on the pilot choice (BookDash vs. ITAN) earlier — ideally before doing the full deep-dive into both issue threads — so research time is spent entirely on the confirmed path rather than comparing two candidates in parallel.

---

## Resources Used

- [docs/ai/imports/index.md, api.md, adding-sources.md, debugging.md](https://github.com/internetarchive/openlibrary/tree/master/docs/ai/imports) (internal Open Library documentation)
- [Issue #8462](https://github.com/internetarchive/openlibrary/issues/8462), [#12091](https://github.com/internetarchive/openlibrary/issues/12091), [#10856](https://github.com/internetarchive/openlibrary/issues/10856), [#8542](https://github.com/internetarchive/openlibrary/issues/8542)
- [BookDash scraper gist](https://gist.github.com/sai-krishna-kotha/2602c9d13b0f3da4c4a0e701096b0219)
- [openlibrary-client import schema](https://github.com/internetarchive/openlibrary-client/blob/master/olclient/schemata/import.schema.json)

---

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
