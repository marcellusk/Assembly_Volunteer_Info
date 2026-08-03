# Project Journal — Assembly Info Builder

A dated log of changes. Newest entries at the top.

## 2026-07-28 — Any-color group picker (schemaVersion 2)

- Replaced the fixed 7-hue palette with a real color picker: 12 curated presets
  plus a native `<input type="color">` for anything else, a hex readout, and a
  live preview chip showing exactly how the group name will render.
- **Data model:** groups now store `color` (`#rrggbb`) instead of `hue` (a
  number from a fixed palette). `migrate()` converts old files via
  `LEGACY_HUES` on every load; group JSON import accepts either. Group-share
  files bumped to `schemaVersion: 2`.
- **Readability is guaranteed, not assumed.** Since users can pick anything,
  `groupUi()` contrast-checks the label and, for colors where neither white nor
  dark text reaches 4.5:1 (vivid pinks/greens), nudges the rendered background
  until one does. The user's chosen `color` is never modified — only the
  rendered value. Fuzz test: 10,000+ random colors, zero contrast failures.
- Discovered while testing that three of my own initial presets needed nudging,
  so the shipped palette was replaced with contrast-safe equivalents; a clicked
  swatch now renders exactly as displayed. Demo Parking orange shifted
  `#b5722a` → `#a46826` as a result.
- Colors resolve once at export into `g.ui {bg, fg, accent}`, consumed by both
  the interactive viewer and the static no-JS view, so they cannot drift apart.
  Removed the last hardcoded hue→hex map from the static view.
- Import hardening: color values land in a `style` attribute, so `normHex()`
  rejects anything that isn't a strict hex (verified against attribute-breakout
  and `javascript:` payloads).
- Regenerated demo, screenshots, and how-to (which now explains the rainbow
  custom-color button and why a color may render slightly adjusted).

## 2026-07-27 — Release v1.0.0 on GitHub

- Published the first tagged release: v1.0.0 (tag on `main` at the iPhone-fix
  commit), with three downloadable assets so non-technical users don't have to
  navigate the repository: the builder, the demo volunteer file, and the
  illustrated how-to guide.
- Release convention going forward: tag `vMAJOR.MINOR.PATCH`, attach those same
  three files (regenerate the demo first if the builder changed), and summarize
  the changes from this journal in the release notes.

## 2026-07-27 — Fix: iPhone "black screen" (no-JS static fallback in exports)

- **Bug:** iPhone users opening the volunteer file from Messages/Mail/Files saw a
  black screen with no errors. Cause: iOS previews HTML attachments with
  QuickLook, which renders HTML but does **not execute JavaScript** — and the
  viewer was 100% JS-rendered on a dark (looked black) background.
- **Fix:** exports now embed a complete static plain-HTML rendering of every
  group (all days, contacts with tel: links, rosters, schedules, notes, images,
  alphabetized directory) plus a banner explaining how to open the interactive
  version. A one-line head script sets `html.js` when JS runs; app layout CSS is
  gated on `html.js`, so with JS the static view is hidden instantly and removed,
  and without JS the file is a normal scrolling light-background document.
- **No size doubling:** image data now lives ONCE, in the static view's `<img>`
  tags; the embedded app JSON carries `#simg-N` markers that the viewer resolves
  from those tags at boot (`buildStaticView()` in the builder produces both).
- Identical-across-days rosters/schedules collapse to a single "Every day"
  block in the static view.
- Verified headlessly with Chrome in both modes (JS on: app identical to before,
  markers resolved, lightbox works; JS off: static view visible, scrollable,
  images load, 63 tel links).
- **Previously distributed exports remain broken on iPhone** — re-export from the
  updated builder and resend. Contributors (human or AI): add an
entry here for every meaningful change, with enough detail that the next person can
understand *why*, not just *what*.

---

## 2026-07-26 — Fictional placeholder maps; first GitHub publish

- Replaced the five real venue map images in the demo seed with generated
  fictional placeholder diagrams (watermarked "FICTIONAL DEMO MAP"), so the
  public repository contains no material derived from the original event
  documentation. Nice side effect: the builder shrank from ~1.6 MB to ~0.33 MB.
- Regenerated the demo export, all screenshots, and README-HowTo.html.
- Published to GitHub (marcellusk/Assembly_Volunteer_Info) with a deny-by-default
  `.gitignore`: nothing is committed unless whitelisted, because the working
  folder also holds real event documents. Pre-commit scan confirmed no real
  names (case-sensitive match against the original roster) and no non-555 phone
  numbers in any committed text file.
- Added `README.md` as the repository landing page.

## 2026-07-26 — README-HowTo user guide with screenshots

- Created `README-HowTo.html`: a self-contained, plain-language user guide for
  non-technical coordinators and captains, with 11 embedded screenshots (double-
  click to open, same as everything else in this project). Covers: the two kinds
  of files (master vs. volunteer handout), event setup, groups, day tabs and
  copy-day, name autocomplete, the directory, images, adding sections, the
  save-your-work habit, exporting to volunteers (with phone-view screenshots),
  the captain JSON workflow, and a common-questions list.
- Raw screenshots live in `docs/screenshots/*.png` (for a future GitHub README).
  They were captured headlessly (puppeteer-core driving installed Chrome against
  the demo seed) — regenerate them after UI changes and re-run the embed step;
  the capture/embed scripts are transient (scratchpad), see HANDOFF.md.

## 2026-07-26 — Pick names from the directory (autocomplete)

- Every Name field in the contacts, roster, schedule, and directory editors now
  offers native autocomplete (`<datalist id="people-list">`) fed by the union of
  **all directory sections across all groups**, deduplicated case-insensitively
  and alphabetized — a volunteer entered in one group's directory is pickable in
  every group (volunteers often serve in multiple departments).
- Picking (or typing) a name that exactly matches a directory entry auto-fills
  the row's Phone field **only if it is empty** — a typed phone is never
  overwritten.
- Free text still works everywhere; the directory is an accelerator, not a gate
  (per the hybrid-registry direction in the architecture discussion).
- The suggestion list refreshes on the persist debounce, so a name typed into a
  directory becomes pickable a moment later without any page action.
- This is the lightweight step toward the shared people registry; entries are
  still stored inline (denormalized), so viewer/export formats are unchanged.

## 2026-07-26 — Per-group JSON export/import (distributed captain workflow)

- New workflow support: instead of one person doing all data entry (or passing the
  master file around serially), each captain can edit their own group in their own
  copy of the builder, click **⇪ Export group JSON** (in the group settings row),
  and send the small `.json` file to the coordinator, who clicks **📥 Import group
  JSON…** (in the Groups card) on the master copy.
- **Images never travel in group JSON** (by design, per the owner's decision — a
  security concern). Export blanks image `src` values; imported image entries
  become "missing — click to attach" placeholders in the builder, and the viewer
  simply skips images that have no src.
- Import is a hard security/validation boundary (`sanitizeSection`): unknown
  section types are skipped, every field is whitelisted and length-capped,
  fully-empty rows are dropped, and any image src that is not a clean
  `data:image/...;base64,` URL is stripped (this closes an HTML-attribute-injection
  vector, since the viewer concatenates img src into markup).
- Day matching on import: exported day ids are mapped to the master's days by id
  first, then by label (case-insensitive), then by position — so a captain who
  renamed "Friday" to "Fri" still imports cleanly.
- Importing a group whose name already exists prompts: replace it, or keep both.
  A summary alert reports sections imported, images to attach, and anything
  skipped or dropped.
- Format: `{format:"assembly-info-group", schemaVersion:1, exportedAt, event:{title,
  days}, group:{…}}` — documented in HANDOFF.md.

## 2026-07-25 — Directories for all demo groups

- Added searchable Volunteer Directory sections to the Cleaning (10), Stage (7), and
  Seating (8) demo groups, so every group now demonstrates the directory feature.
  All entries fictional; a few include constraint examples.
- Regenerated `Demo Regional Convention - Volunteer Info.html`.
- Note for future work: per-group directories now visibly duplicate people who also
  appear in that group's rosters/schedules — supporting evidence for the planned
  shared people registry (see HANDOFF.md roadmap / architecture discussion).

## 2026-07-25 — Demo data, documentation, GitHub prep

- Replaced all real names and phone numbers in the builder's seed data with fictional
  ones (phone numbers use the reserved `555-01xx` fictional range).
- Expanded the seed from one group to a full four-group demo: **Parking, Cleaning,
  Stage, Seating** — chosen to exercise every section type and feature:
  - Parking: contacts, two rosters (morning + afternoon), schedule, shared notes,
    5 embedded maps, 13-entry directory with constraint examples.
  - Cleaning: zone-based roster, midday rotation schedule.
  - Stage: **per-day notes** (different text Fri/Sat/Sun) to demo that toggle.
  - Seating: minimal group (contacts, roster, notes) to show sections are optional.
  - Saturday rosters differ slightly from Friday/Sunday (extra person in Parking
    Lot #2) to demo per-day editing.
- Event retitled "Demo Regional Convention"; footer notes the data is fictional.
- Regenerated the sample export as `Demo Regional Convention - Volunteer Info.html`
  and deleted the earlier sample that contained real data.
- Created this journal and `HANDOFF.md`.
- **Not safe for GitHub:** the legacy `Parking Info - standalone*.html` files and any
  saved project/export files made from real data still contain real names and phone
  numbers. Only publish `Assembly Info Builder.html`, the demo export, `JOURNAL.md`,
  and `HANDOFF.md` (add everything else to `.gitignore`).

## 2026-07-25 — Initial build

- Built `Assembly Info Builder.html`: a single-file, no-install utility (vanilla JS,
  no dependencies, no build step) that creates and edits multi-group volunteer info
  for assembly events and exports self-contained viewer HTML files.
- Design decisions (made by the project owner):
  - **Builder + viewer-only exports** — the builder file is the editing master;
    exported files are locked/view-only (no edit mode for volunteers).
  - **Flexible section building blocks** per group rather than a fixed template:
    Contact List, Assignment Roster (per-day), Time-Slot Schedule (per-day), Notes
    (shared or per-day), Images/Maps, Volunteer Directory.
  - **Auto-compressed images** on attach (max 1600 px, JPEG q0.85; keeps original if
    smaller) so exports stay small enough to text/email.
  - **Swipe + top pill bar** navigation in the viewer: swipe left/right between
    groups, global Fri/Sat/Sun day tabs, tap-to-call phone links, tap-to-zoom images.
- Persistence: localStorage autosave + "Save Project File" (the builder clones its
  own DOM with the project JSON embedded in `#project-data`). Restore-banner conflict
  handling guarded by `projectId`. "New Event" starts a blank project.
- Migrated the complete 2026 Regional Convention parking department data (rosters for
  three days, rotation, directory, five maps) out of the old hand-edited
  `Parking Info - standalone.html` (a "bundler"-wrapped file whose real content lived
  in `__bundler/template` + `__bundler/manifest` script tags) into the new format.
- Fixed en route: `String.replace` `$`-pattern corruption risk in export (use function
  replacements); one map image had a mislabeled `image/png` MIME on JPEG bytes.
