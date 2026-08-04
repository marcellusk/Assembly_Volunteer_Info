# Project Journal — Assembly Info Builder

A dated log of changes. Newest entries at the top.

## 2026-08-03 — Navigation for the no-JavaScript (iPhone) view

- Measured the gap first: the static view was 13 phone screens of continuous
  scroll for the demo (7 for the single-department Lunch Delivery example) with
  **zero** in-page links and no way to skip a day — and that is what most
  iPhone volunteers see, since iOS previews HTML attachments without scripts.
- Added navigation built only from HTML/CSS, so it works with scripts off:
  - a **department jump row** at the top and a **day jump row** inside each
    department (plain `#fragment` anchors);
  - each day is now a `<details open>` block with the date as its summary, so
    days can be folded away.
- **Everything is open by default and no CSS-only tab tricks were used.** If a
  previewer refuses to toggle `<details>` or follow anchors, the page is still a
  complete readable scroll — the fail-safe rule the static view exists for.
- Restructured the static view around **days rather than sections**: content
  identical on every day renders once (labelled "same every day") instead of
  being repeated, and only day-varying sections sit inside the day blocks.
  Author ordering is preserved by splitting sections before/after the first
  day-varying one, so contacts stay on top and maps/directory at the bottom.
- Maps now render two-up and height-capped; at full width a handful of them
  buried everything below. Pinch-zoom still reads them.
- Verified with JavaScript disabled: department and day anchors land correctly,
  tapping a day summary collapses it, and folding all days took the demo from
  13 screens to 8. The interactive path is unchanged (image markers still
  resolve, lightbox and swipe unaffected).
- **Still unverified:** whether iOS QuickLook permits link taps and `<details>`
  toggling at all. Needs a physical-device check; noted in HANDOFF.md.

## 2026-08-03 — Calendar dates on days; viewer opens on today

- Days now carry an optional ISO `date` alongside their label. The builder gives
  each day a date picker; **picking a date writes the label** ("Fri, Jul 9") and
  the label stays editable afterwards. A "Fill dates from the first day" button
  gives the remaining days consecutive dates, since most events run back to back.
- **The exported file opens on today.** If the device's date matches one of the
  event's dates, the viewer starts on that day's page (header and swiper both);
  otherwise it starts on the first day, exactly as before.
- **All date handling is local-time.** `new Date("2027-07-09")` parses as UTC and
  `toISOString()` reports UTC, either of which shows the wrong day west of
  Greenwich in the evening — precisely when volunteers check the schedule.
  Dates are built from `getFullYear/getMonth/getDate` and parsed via
  `new Date(y, m-1, d)` throughout. Regression-tested at 11:30 PM Sunday in
  America/Chicago (04:30 UTC Monday): still opens on Sunday.
- Import now matches days **by date before label**, so two files covering the
  same event agree even when one says "Fri" and the other "Friday". Days added
  during import carry their date across.
- Day tabs scroll horizontally and no longer shrink, since date labels are
  longer than "Friday" and an event can run Monday–Thursday.
- Backward compatible: days without a date behave exactly as before (no
  auto-open, label shown as typed). Demo seed and the Lunch Delivery example
  were given real dates.

## 2026-08-03 — Import can add days the event doesn't have; Lunch Delivery example

- Added `Lunch Delivery - group.json`, a fictional example import covering a
  **Monday–Thursday** work week: dish assignments (each person's row carries the
  actual dish in the role field), serving teams, cleanup crews, a kitchen
  timeline, per-day menu notes, image placeholders, and a 19-person directory.
  Doubles as documentation of the group-JSON format for other tools.
- **Import now handles days the event doesn't have.** Previously unmatched days
  fell back to positional matching, so a Mon–Thu file dropped into a Fri–Sun
  event silently landed Monday's data on Friday. `buildDayMap()` now returns
  unmatched days instead of guessing, and the import asks: add them as their own
  days, or match them in order to the existing ones. Adding a day also gives
  every existing section an empty slot for it (`addEventDay`).
  - Verified: Mon–Thu into a Fri–Sun event with "add" produces 7 days with data
    intact; a captain file that merely renamed days (Fri/Sat/Sun) with "match in
    order" maps correctly and adds nothing; exact label matches
    (case-insensitive) never prompt at all.
- Schedule rows wrap to a second line when they carry both a phone number and a
  WhatsApp button, instead of squeezing the name — noticed while reviewing the
  Lunch Delivery preview on a phone-width viewport.
- Confirmed the section types cover this shape without changes: a roster's areas
  work as "Bringing Dishes / Serving Team / Cleanup Crew" and the role field
  carries "Baked ziti — 2 large pans". Schedule rows inherited WhatsApp from the
  directory automatically (28 buttons across the four days, none typed twice).

## 2026-07-28 — Schedules inherit WhatsApp from the directory

- Schedules already rendered a WhatsApp button when the row carried a value, but
  that meant retyping the number for every rotation slot. Now a contacts or
  schedule row with no WhatsApp of its own **inherits it from that person's
  volunteer-directory entry**, so entering it once in the directory makes the
  button appear everywhere that person is listed — including retroactively for
  rows created before the directory had it.
- Implemented as `resolveContactLinks()` inside `buildStaticView`, so the
  interactive viewer and the static no-JS view get identical results from one
  code path, and it applies to the **export copy only** — the editor's stored
  data is never rewritten behind their back.
- A row's own value always wins, so a person can have a different WhatsApp
  number in a specific slot. Matching is exact on the trimmed, case-insensitive
  name across every group's directory.
- The editor shows a `from directory: 816-555-0101` placeholder on rows that
  will inherit, so the behavior is visible rather than magic; suppressed on
  directory rows themselves since that is where the value comes from.
- Verified in the demo: Alan Booker's directory WhatsApp now appears on the
  Friday and Sunday rotation rows with nothing typed there, Victor Ramos
  correctly shows his *different* WhatsApp number rather than his phone, and
  people without WhatsApp still show no button. Static view count rose 9 → 12.

## 2026-07-28 — Swipe now moves between DAYS, not departments

- Reported as "lost the ability to swipe left/right on Android". Investigated
  first: swipe was **not** broken — verified on emulated Android (Pixel 7, real
  touch events) that pure-horizontal, diagonal (30px/60px drift), and
  post-vertical-scroll swipes all worked, and the layout CSS was unchanged since
  the last known-good release. The actual issue was an expectation mismatch: the
  gesture was bound to departments while the owner (and therefore volunteers)
  expected days.
- **Changed the swipe axis**: the swiper now holds one page per *day of the
  current department*. Swiping moves Friday → Saturday → Sunday; departments are
  picked from the pill bar, which rebuilds the pages and keeps the current day.
  This also fixes single-department files, where department-swiping had nothing
  to move to (verified: 1 group previously produced one page and no swipe).
- `renderSection` now takes `dayId` explicitly rather than reading a global,
  since sibling pages render different days.
- Non-per-day sections (contacts, images, directory) repeat on each day page, so
  viewer images gained `loading="lazy" decoding="async"` to avoid decoding the
  same maps three times — relevant to the older-iPhone memory concern.
- Verified: swipe both directions updates the day tab; tapping a day tab jumps;
  switching department preserves the day; each page holds the correct day's data
  (Saturday-only roster entry and per-day Stage notes appear only on their own
  page); directory open/search shared across a department's day pages; lightbox
  and the no-JS static view unaffected.
- Updated the in-app hint, the how-to guide, and regenerated screenshots.

## 2026-07-28 — WhatsApp contacts alongside phone numbers

- Volunteers on site often reach each other over WhatsApp because plain calls
  get ignored as spam. Contacts, schedule rows, and directory entries now carry
  an optional `wa` field; the viewer shows a green **WhatsApp** pill next to the
  phone number so either can be tapped. Works in the static no-JS view too, so
  iPhone attachment previews get it as well.
- Editors gained a "WhatsApp (number or link)" box plus a small `=` button that
  copies the phone number across (the common case). **Empty means the person has
  no WhatsApp** and no pill is shown — deliberately not auto-derived from the
  phone, since a wa.me link to a non-WhatsApp number is a dead end.
- Accepts a number in any formatting or a pasted wa.me / whatsapp.com link.
  `waLink()` normalizes; 10-digit numbers get a `1` prefix (same US assumption
  as the existing `tel:` links).
- Security: the value lands in an `href`, so only wa.me / whatsapp.com URLs or
  plain numbers are accepted — verified against `javascript:`, non-TLS `http:`,
  and lookalike hosts (`wa.me.evil.com`, `evil.com/wa.me/`, `wa.me@evil.com`).
  Import strips bad values and reports the count; the viewer re-validates rather
  than trusting the exported JSON.
- Name autocomplete now fills WhatsApp as well as phone from the directory,
  correctly preserving a WhatsApp number that differs from the phone number.
- Pill uses a generic speech-bubble icon, not the WhatsApp mark, so no
  third-party logo ships in the repository.

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
