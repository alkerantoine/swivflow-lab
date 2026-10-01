# SwivFlow LAB → Production Port Notes — 2026-10-01

All changes below are **LAB ONLY** (`~/workspace/swivflow-lab/wp-content/plugins/swivflow/`).
Production (1.29.39 line) is untouched. Port each change deliberately; nothing here
auto-syncs.

Relevant user decisions baked in:
- The date entered in New Schedule is authoritative for that launch; AI Planner must
  not ask for it again.
- Branding stays two-tone **Swiv** (theme text) + **Flow** (`#f09030`); no square "S" tile.
- Inter everywhere; Space Grotesk never reintroduced.
- Light mode is attribute-less: `html:not([data-theme="dark"])`, never `html[data-theme="light"]`.
- No `!important` inside custom-property values.

---

## 1. Mobile Lower Third — Prepared rows are tap-to-stage, feeding SHOW

**File:** `assets/manager.html` (inline script, mobile IIFE scope)

**Mechanism:**
- New module-scope state: `stagedMobileLtTemplateId` (string template id, or `null`).
- `renderMobileLt()` renders the Prepared list as real `<button>` elements:
  `<button class="m-lt-tpl-btn" data-lt-tpl="<template-id>" aria-pressed="true|false">`.
  The staged row additionally carries the `.staged` class.
- Tapping a row toggles staging (tap staged row again = unstage). Row classes and
  `aria-pressed` update in place — the list is NOT rebuilt, so focus and DOM handles
  survive.
- `renderMobileLtPreview()`: when a template is staged, the preview shows a staged
  card reading `Staged · press SHOW to put on air` naming the staged template.
- Mobile SHOW routes through new `showStagedMobileLtTemplate()`:
  1. Early-outs (unchanged live state): mutation not ready → `liveLowerThirdNotReady()`;
     no live schedule → toast `Set a schedule Live first.`; runtime missing →
     toast `Lower Third engine unavailable.`
  2. Builds the payload for the STAGED template directly, with zero live-state writes:
     `template = rt.findTemplate(state, String(tplId))`, then
     `payload = rt.makeInstance(state, schedule, index, template, String(tplId))`,
     the whole construction wrapped in try/catch → `payload = null` on any throw.
     `rt` is `window.SwivFlowLowerThird12916`. Deliberately does NOT go through
     `rt.resolve()` and does NOT touch `state.outputs.lowerthirdLiveTemplateId`
     beforehand (an earlier LAB iteration temporarily mutated that field; removed).
  3. If no payload: toast `No Lower Third is available for this segment.` and return —
     live output untouched.
  4. Only after a payload exists, writes `state.outputs.lowerthirdProgramPayload`,
     `state.outputs.lowerthirdLiveTemplateId` (from `payload.templateId`), sets
     `lowerthird=true`, recomputes `master`, then `outputRefreshFrames('lowerthird')`,
     `persist({refreshScope:'lowerthird'})`, `renderLowerThirdMutationUi()`.
  5. Clears `stagedMobileLtTemplateId`, re-renders mobile LT, toast `Lower Third shown.`
- HIDE: existing `hideMobileLt()` hides output; staging is preserved.

**CSS (same file, existing around the mobile LT styles):**
`#mLt .m-lt-tpl-btn`, `#mLt .m-lt-tpl-btn.staged` (blue border highlight),
`#mLt .m-lt-staged` (dashed staged preview card).

**Failure-safety contract (unit-tested in Node against the extracted function):**
not-ready, no-schedule, no-runtime, findTemplate-throws, makeInstance-throws,
makeInstance-returns-null → all six leave `state.outputs` byte-for-byte identical
and surface a toast. Success path sets `lowerthird=true` with the staged template id.
Because payload construction is synchronous with no `persist`/`render` between guard
and commit, no observer can see an intermediate state.

**Selectors for porting tests:** `#mLtTemplateList [data-lt-tpl]`, `#mLtPreviewWrap`,
`#mLtShow`, `#mLtHide`, `button[data-tabpage="lowerthird-designer"]`.

---

## 2. New Schedule → AI Planner: deterministic date + name carryover (mobile + desktop)

**Sender — `assets/manager.html`, shared `createSchedule()` (both viewports):**
When Starting point = "Start with AI planner", navigates to
`/app/ai-assist/?org=<org>&eventDate=YYYY-MM-DD&scheduleName=<name>` where
- `eventDate` is appended only if it matches `/^\d{4}-\d{2}-\d{2}$/`,
- `scheduleName` is the trimmed entered name,
- existing `org` param preserved.

**Receiver — `assets/ai-assist.js`:**
- Reads `eventDate`/`scheduleName` from the launch URL on load.
- Strict ISO date validation incl. real-calendar-date check; name trimmed, capped at
  140 chars.
- Seeds `state.foundationAnswers = carriedPlanDate ? { date: carriedPlanDate } : {}`,
  so `pendingFoundation()` skips the date question entirely. Existing `buildPlan()`
  precedence (`a.date || f.date`) unchanged.
- Shows an informational confirmation banner:
  `Planning for <name> · <Month D, YYYY> — carried from your New Schedule form. I won't ask for the date again.`
- After `swivflow_ai_analyze` returns, overwrites the proposal title with the carried
  name (`state.current.title` and nested `state.current.proposal.title` when present).
- Strips `eventDate`/`scheduleName` from the URL via history replace after capture
  (keeps `org`), so refresh/back cannot reuse consumed launch seeds.

**Cache version — `includes/class-swivflow-ai-assist.php`:**
AI Assist JS `1.29.16 → 1.29.36`. CSS version untouched (no CSS change).

**Deliberate non-change:** server foundation filter in
`includes/class-swivflow-ai-planning.php` (`analyze()`, ~line 608) still accepts only
`date, startTime, endTime, timingNote, timingUnknown`. Title enforcement is client-side
deterministic; do NOT add `scheduleName` to the server foundation without a separate
decision (existing APIs must keep working).

---

## 3. Inter typography (mobile + desktop)

**File:** `assets/manager.html`
- Removed the three Space Grotesk `@font-face` blocks.
- `.m-card h3`, `.lt-title`, `.lt-next-title` now render in Inter.
- Physical Space Grotesk font files deliberately retained on disk (not deleted).

---

## 4. Mobile bottom-sheet modals (≤760px)

**File:** `assets/manager.html` CSS:
```css
.modal-bg{padding:0!important;align-items:flex-end!important;z-index:800!important}
.toast{z-index:800!important}
```
All modals (New Schedule, Add Segment, etc.) dock to the bottom with 16px top radius;
toasts render above the sheet.

---

## Browser verification (LAB, 2026-10-01)

Chromium 152 via local Playwright through the Pinggy tunnel
(`https://cqhvt-2a04-4e41-4-d32d--6799-d32d.free.pinggy.net`), fresh logins.

**Mobile 390×844 — Lower Third (10/10):** 10 Prepared rows render as buttons; "Salem
Speaker" stages (`aria-pressed=true`, `.staged`); preview shows staged card; tap
toggles off; SHOW puts exactly the staged template on air (toast `Lower Third shown.`);
HIDE hides output (`Nothing on air`); zero console/page errors.

**Mobile 390×844 — final (8/8):** "6 pou 6 — Fasting & Prayer" computed font-family
starts with Inter; Add Segment modal opens as a true bottom sheet
(`align-items:flex-end`, `padding-top:0`, 16px top radius) with reachable Close;
Save Segment adds the segment (toast `Segment added.`, appears in list); probe segment
deleted afterward; zero page errors. Note: mobile Plans has no Add Segment affordance
(segment list is read-only on mobile); the shared `#segmentModal` was exercised at the
mobile viewport.

**Desktop 1280×800 — AI parity (7/7):** New Schedule → Start with AI planner with
name `Desktop Parity Check`, date `2026-11-20` → AI URL carries exact
`eventDate=2026-11-20&scheduleName=Desktop+Parity+Check`; banner reads
`Planning for Desktop Parity Check · November 20, 2026 — carried from your New
Schedule form. I won't ask for the date again.`; launch params stripped to
`/app/ai-assist/?org=1` after capture. (Mobile run earlier carried
`eventDate=2026-10-15&scheduleName=Verify+AI+Date+Carryover` with identical banner
behavior.)

**Failure-safety unit tests (7/7):** extracted `showStagedMobileLtTemplate` run under
Node with stubbed dependencies — all six failure scenarios leave `state.outputs`
untouched; success path commits correctly.

**Source checks:** all 3 inline `manager.html` scripts + `ai-assist.js` pass
`node --check`; `html[data-theme="light"]` count = 0; active Space Grotesk usage = 0;
no `!important` inside custom-property values.

**Still not covered (known gaps, not regressions):** full end-to-end AI planning run
proving no second date question and exact title through `swivflow_ai_analyze`
response; refresh/back seed-consumption across a completed run; 320px widths.

## Status
- Implementation: complete (LAB).
- Browser verification: complete (LAB).
- User acceptance: pending.
