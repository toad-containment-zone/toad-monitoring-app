# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A no-build, offline-first field data collection tool for cane toad Time-to-Detection surveys (Toad
Containment Zone / TCZ program): field crews visit waterpoint sites across a landscape, time how
long it takes to detect a toad, and sync the records to an ODK Central form. It is a sibling app to
`../goanna-hunting-app` (same TCZ program, same no-build single-HTML-file philosophy, same
OpenRosa-only interfacing with ODK Central) but with one structural difference: **sites are
revisited across years**, so they're modelled as **ODK Central Entities** rather than being
captured fresh on every submission. Read `../goanna-hunting-app/CLAUDE.md` first if you haven't —
large parts of this app (settings/sync engine, timer, QR scanning, PWA shell) are a direct port of
that app's patterns, and this file mostly documents what's *different*.

- `toad-monitoring-app.html` — the entire application: HTML, CSS, and vanilla JS in one file, no
  dependencies, no bundler, no package.json.
- `toad_detection_survey.xlsx` — the companion XLSForm (sheets: `survey`, `choices`, `entities`,
  `settings`) defining the ODK Central form that the app submits to, including the `toad_monitoring_sites`
  entity dataset declaration. Field names in this workbook must line up with the hand-built XML in
  `buildSubmissionXml()` in the HTML's `<script>` — if one changes, the other needs to change too.
  **This file was compiled and validated locally** with `pyxform`/ODK Validate (`xls2xform`) to
  confirm it produces a legal XForm — see "Entities: what's verified vs. assumed" below before
  trusting anything beyond that.
- `manifest.json` / `sw.js` — web app manifest and service worker that make the app installable
  and precache the shell for offline use. Same mechanism as the goanna app; see its CLAUDE.md's
  "PWA shell" section.
- `index.html` — a bare redirect to `toad-monitoring-app.html`, so the site's root URL works
  without visitors needing to know or type the actual filename. No app logic of its own.

There is no build step, package manager, linter, or test suite. "Running" the app means serving
the directory statically (e.g. `python3 -m http.server`) and opening `toad-monitoring-app.html` —
the service worker requires a real HTTP(S) origin (or `localhost`), so a static server is needed to
test offline/install behaviour. Verify changes by opening it in a browser and exercising the flow.

If you edit `toad_detection_survey.xlsx`, re-validate it rather than trusting the workbook alone:
```
pip install pyxform   # in a throwaway venv — don't touch the system Python
xls2xform toad_detection_survey.xlsx /tmp/check.xml
```
A clean "Conversion complete!" means pyxform and ODK Validate both accepted it. This project has no
live ODK Central instance to test submissions against, so this compile check is the only automated
guardrail there is — treat a validation failure as a real bug, and treat a pass as "structurally
legal," not "definitely matches what Central expects at runtime."

## Architecture

Organized into commented sections in the single `<script>` block (search for `// ---------- ...
----------` markers), following the same shape as the goanna app:

1. **Settings & persistence** — `settings` (sync config only: `appUserCode`/`submissionUrl`, via
   the same ODK Central app-user QR-code decoding as the goanna app — see its CLAUDE.md for how
   `resolveSubmissionUrl()` works) and `records` (survey history), persisted to `localStorage`
   under `tcz_toad_settings_v1` / `tcz_toad_records_v1`. A third store, `tcz_toad_sites_cache_v1`,
   holds the last-fetched list of known sites (see below). Unlike the goanna app, there's no global
   team-name or native-title-area setting — native title area is a **site** property here (sites
   span a landscape, so it varies per site, whereas goanna hunts happen in one contained area per
   session).
2. **Known sites (ODK Entities)** — this is the part with no goanna-app precedent. Sites are
   registered once and reused across years, so they're the `toad_monitoring_sites` entity dataset
   declared on `toad_detection_survey.xlsx`'s `entities` sheet. The app needs an offline-capable way
   to (a) list sites already known to Central and (b) register new ones from the field — both using
   only the OpenRosa endpoints an ODK Central **app user** token can reach (formList/manifest/
   attachments/submission), the same auth model the goanna app already relies on, deliberately
   *not* Central's separate, higher-privilege REST Entities API:
   - **Listing**: Central auto-publishes a `toad_monitoring_sites.csv` media file attachment on any
     form whose XLSForm references that dataset (this form's `site_id` question is a
     `select_one_from_file toad_monitoring_sites.csv` for exactly this reason, even though the
     app never renders it as a native select widget — its only job is to make Central treat this
     form as a consumer of the dataset and attach the CSV). `fetchSiteList()` fetches the form's
     `/manifest` (OpenRosa XML), finds the `<mediaFile>` whose filename matches, downloads that CSV,
     and parses it with a small hand-rolled `parseCsv()`. Results are cached in `sitesCache`
     (`{ fetchedAt, sites }`) and refreshed opportunistically (opening the site sheet, if the cache
     is >5 minutes old and online) or on demand ("Refresh site list" button).
   - **Registering**: a new site isn't submitted separately — it's captured in the *same*
     submission as the first survey against it (`is_new_site = 'yes'` gates both the visible "new
     site" form fields and the entity's `create_if`). `allKnownSites()` merges the last-fetched CSV
     with any site this device has registered locally (from `records` where `is_new_site='yes'`,
     regardless of sync status) so a freshly-registered site is selectable for a second visit
     immediately, without waiting for the next manifest refresh — tagged "(pending sync)" in the
     picker until the record actually syncs.
   - **Never updating**: `update="false"` is hardcoded in every submission's `<meta><entity>`
     element. Reusing an existing site leaves its property fields unanswered/blank in that
     submission; without `update="false"` an upsert would overwrite the entity's real properties
     with those blanks. See `entities` sheet: `update_if` is `false()` for the same reason —
     pyxform requires an explicit `update_if` whenever both `entity_id` and `create_if` are present
     (this "upsert" detection tripped the compile check once while building this; the fix was
     adding `update_if=false()`, not removing `entity_id`).
3. **Site selection UI** — the site-select bottom sheet (`#siteOverlay`) is the app's entry point,
   replacing the goanna app's "Start hunting" button. `openSiteSheet()` shows either an
   existing-site picker (search + tap-to-select from `allKnownSites()`) or a new-site registration
   form (lat/lon — GPS-prefilled *but manually editable*, unlike the goanna app's GPS-only capture,
   because site coordinates are entered once and reused for years so it's worth letting a crew
   correct a bad GPS fix; site name; waterpoint type checkboxes; natural/artificial toggle; native
   title area). Confirming stores the choice in `currentSite` (in-memory only — nothing is
   submitted yet) and reveals the timer card. There's no persistent "session" wrapper like goanna's
   hunting session: each site visit is one standalone submission, matching the Fulcrum data model
   this app replaces (see "Fulcrum migration" below).
4. **Timer** — same start/stop stopwatch pattern as the goanna app (`running`, `startTs`,
   `tick()`), relabelled: "Start search" / "Stop search". `time_searched_minutes` is the elapsed
   time rounded to the nearest minute (Fulcrum's original field granularity), not seconds. Unlike
   the goanna app, stopping the timer doesn't imply detection — "was a toad found?" is asked
   explicitly in the record sheet afterwards, since a search can end with no detection at all.
5. **Record lifecycle** — same `pending → synced`/`failed` shape as the goanna app, one record per
   site visit (no multi-record "session" to finish). Field names on the record object are kept
   **identical to the XLSForm survey sheet's `name` column** (no separate `FIELD_MAP` translation
   layer like the goanna app has) — deliberately, to remove one source of drift between the two
   files; if you rename a field in one, rename it in the other.
6. **XForms/OpenRosa submission building** — `buildSubmissionXml()` emits a flat instance (no XML
   groups, matching the goanna app's precedent — this app hand-builds submission XML rather than
   running an XForms engine, so keeping the instance flat means the JS's own branching logic can
   stay the source of truth instead of having to mirror `relevant`/`create_if` expressions at
   runtime) plus a `<meta><entity>...</entity><instanceID>...</instanceID></meta>` block. **The
   element order and the entity block's shape were copied from pyxform/ODK Validate's actual
   compiled output for this XLSForm**, not written from memory — see "Entities: what's verified vs.
   assumed" below.
7. **Sync orchestration**, **QR code scanning** — unchanged from the goanna app; see its CLAUDE.md.

### Entities: what's verified vs. assumed

Verified by locally compiling `toad_detection_survey.xlsx` with `pyxform`/`xls2xform` (ODK
Validate passed): the XLSForm itself is structurally legal, and the instance XML shape
`buildSubmissionXml()` produces (element order, the `<meta><entity dataset="..." create="..."
update="false" baseVersion="" trunkVersion="" branchId="" id="...">` attributes, the `<label>`
child) matches what that XLSForm compiles to.

**Not verified** (no live ODK Central project was available while building this) — check these
against a real Central project before relying on the app in the field:
- That Central actually auto-attaches `toad_monitoring_sites.csv` to this form's manifest for
  **app users** specifically (vs. only for web-user/Collect-authenticated sessions) — this may
  require an explicit admin step in Central's project settings that isn't visible from the
  XLSForm alone.
- That posting `baseVersion=""`/`trunkVersion=""`/`branchId=""` (rather than omitting the
  attributes, or some other placeholder) is accepted for a plain reference (`update="false"`) to
  an existing entity — these attributes exist for conflict-safe *updates*, which this app never
  does, but Central's submission parser may still validate their presence/format.
- That creating an entity via a bare OpenRosa `xml_submission_file` POST (no ODK Collect involved)
  is accepted the same way it would be from Collect — the entities feature was built and is
  primarily tested against Collect's submission path.

If any of these turn out to be wrong, the fix is almost certainly local to `buildSubmissionXml()`
and/or the `entities` sheet — the rest of the app (sync queue, offline storage, timer) doesn't need
to know or care how entity creation works under the hood.

### PWA shell (`manifest.json`, `sw.js`, `index.html`)

Identical mechanism to the goanna app — see its CLAUDE.md. `CACHE_NAME` here is
`tcz-toad-shell-v1`; bump it whenever you change what needs to be cached.

### Styling

Same CSS-custom-properties approach as the goanna app, deliberately re-themed rather than reusing
its exact palette so the two apps are visually distinguishable at a glance in the field (crews may
have both installed): swamp-green/clay/cream (`--marsh`, `--clay`, `--rock`, `--cream`) instead of
goanna's olive/dirt. Same single-column mobile shell, same bottom-sheet modal pattern.

## Shared taxonomy

Native title area is a site property (not a global setting — see above). The `nta_choices` list in
`toad_detection_survey.xlsx` and the `#newSiteNta` `<select>` in the HTML are a hand-copied snapshot
of the *active* rows in `../shared-taxonomy/taxonomy/native_title_areas.csv` (currently: KJ, NYKJ,
NY, YWR, NML) — that repo is the authoritative source; re-sync both places if it changes. The
submitted value is the NTA's mnemonic `nta_id`, not a display label, so it joins directly against
that table downstream. (The goanna app's own copy of this list has drifted — its `#cfgNta` dropdown
is missing YWR/NML even though its CLAUDE.md claims otherwise — this app's copy was taken directly
from the current CSV to start clean.)

## Fulcrum migration

Historical data was collected in Fulcrum before this app existed. See `FULCRUM_MIGRATION.md` for
the field mapping from Fulcrum's site/survey schema to `toad_monitoring_sites` entity properties
and this form's survey fields, and notes on importing it into ODK Central. That document is a plan,
not a working import script — no Fulcrum export was available while drafting this app.

## Notes for editing

- Keep the *application* in the single `toad-monitoring-app.html` file — no dependencies, no
  bundler — unless explicitly asked to split it up. Same exception as the goanna app for
  `manifest.json`/`sw.js` (service workers can't be inlined).
- The vendored jsQR block and the base64 app icon are embedded inline exactly as in the goanna app
  — leave them alone unless the logo/QR library itself needs to change. The icon here is a
  generated water-drop mark (Pillow-drawn), not hand-designed artwork — regenerate rather than
  hand-edit the base64 if it needs to change.
- `crypto.randomUUID()` generates both `site_id` (which doubles as the entity's real key when
  `is_new_site='yes'`) and the OpenRosa `instanceID` — don't change ID generation without
  preserving uniqueness guarantees, and don't conflate the two IDs even though the same function
  produces both.
