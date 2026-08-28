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
Don't just double-click the file open as a `file://` URL: besides the service worker not working
there, the map view's OpenStreetMap basemap tiles will 403 ("referer is required by tile usage
policy") since `file://` pages send no `Referer` header — the same static-server setup fixes both.

If you edit `toad_detection_survey.xlsx`, re-validate it rather than trusting the workbook alone:
```
pip install pyxform   # in a throwaway venv — don't touch the system Python
xls2xform toad_detection_survey.xlsx /tmp/check.xml
```
A clean "Conversion complete!" means pyxform and ODK Validate both accepted it. This compile check
is the only *automated* guardrail in this repo, so treat a validation failure as a real bug and a
pass as "structurally legal," not "definitely matches what Central expects at runtime." A live
Central project now exists (the Fulcrum migration uploaded historical sites/submissions to it —
see "Entities: what's verified vs. assumed" below and `FULCRUM_MIGRATION.md`), which has verified
some runtime behaviour empirically, but that was exercised via web-user REST calls from migration
scripts, not this app's own app-user OpenRosa path — see that section for exactly what remains
unverified end-to-end through the app itself.

## Architecture

Organized into commented sections in the single `<script>` block (search for `// ---------- ...
----------` markers), following the same shape as the goanna app:

1. **Settings & persistence** — `settings` (sync config only: `appUserCode`/`submissionUrl`, via
   the same ODK Central app-user QR-code decoding as the goanna app — see its CLAUDE.md for how
   `resolveSubmissionUrl()` works) and `records` (survey history), persisted to `localStorage`
   under `tcz_toad_settings_v1` / `tcz_toad_records_v1`. A third store, `tcz_toad_sites_cache_v1`,
   holds the last-fetched list of known sites (see below). Unlike the goanna app, there's no global
   team-name setting. Native title area and pastoral property were once captured here as per-site
   fields, but the app no longer collects them — the data warehouse's ETL now derives both from
   each record's geolocation (spatial join against NTA / station boundaries), so the reporting
   layer is the single source of truth. See "Removed: native title area & property" below.
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
     submission as a survey record (`is_new_site = 'yes'` gates both the visible "new site" form
     fields and the entity's `create_if`). There are two ways that submission comes about: (a) the
     normal end-of-visit flow, where the crew searches a site, records survey results, and only then
     picks "register new site" as part of choosing which site the just-finished search belongs to
     (`survey_conducted='yes'` on that record); or (b) the standalone "Register a new site" shortcut
     on the main screen, for crews who establish a site during the day and don't run a timed search
     until a later visit — this submits immediately with no timer/survey step involved, and
     `survey_conducted='no'` flags it so it's never mistaken for a real zero-effort search
     downstream (all five survey-outcome fields — `temperature_c`, `toads_access_water`,
     `people_searching`, `time_searched_minutes`, `toad_found` — are left blank on these records; the
     XLSForm's `relevant`/`required` on those fields are conditioned on `${survey_conducted} =
     'yes'`, matching the `is_new_site`-conditioned site fields above it). `allKnownSites()` merges
     the last-fetched CSV with any site this device has registered locally (from `records` where
     `is_new_site='yes'`, regardless of sync status or which of the two paths created it) so a
     freshly-registered site is selectable immediately, without waiting for the next manifest
     refresh — tagged "(pending sync)" in the picker until the record actually syncs.
   - **Never updating**: `update="false"` is hardcoded in every submission's `<meta><entity>`
     element. Reusing an existing site leaves its property fields unanswered/blank in that
     submission; without `update="false"` an upsert would overwrite the entity's real properties
     with those blanks. See `entities` sheet: `update_if` is `false()` for the same reason —
     pyxform requires an explicit `update_if` whenever both `entity_id` and `create_if` are present
     (this "upsert" detection tripped the compile check once while building this; the fix was
     adding `update_if=false()`, not removing `entity_id`).
3. **Timer** — `#timerCard` is the app's entry point (visible on load, no site or anything else
   required first) — field crews arrive at a site and start searching immediately, so the very
   first thing they interact with is "Start search"/"Stop search", the same start/stop stopwatch
   pattern as the goanna app (`running`, `startTs`, `tick()`). `time_searched_minutes` is the
   elapsed time rounded to the nearest minute (Fulcrum's original field granularity), not seconds.
   Unlike the goanna app, stopping the timer doesn't imply detection — survey results (including
   "was a toad found?") are asked explicitly afterwards, since a search can end with no detection
   at all.
4. **Survey results, then site selection** — stopping the timer opens the survey-results sheet
   (`#overlay`: people searching, temperature, water access, toad found, notes) *before* the app
   knows which site this was. Tapping "Next: choose site" stashes those answers in a
   module-level `pendingSurvey` object (nothing's saved yet) and opens the site-select bottom sheet
   (`#siteOverlay`, `openSiteSheet('finalize')`) — either an existing-site picker or a new-site
   registration form (lat/lon — GPS-prefilled *but manually editable*, unlike the goanna app's
   GPS-only capture, because site coordinates are entered once and reused for years so it's worth
   letting a crew correct a bad GPS fix; site name; waterpoint type checkboxes; natural/artificial
   toggle). Confirming builds `currentSite` (via `confirmExistingSite()`
   for the existing-site path, shared by every UI that can produce a site id — the list row click and
   the map marker popup's button both funnel through it), merges it with `pendingSurvey`, and *that's*
   the point the record is actually saved/queued (`finishSiteConfirmation()`) — the app then resets to
   a fresh idle timer for the next visit, and a brief auto-dismissing toast (`showToast()`, the only
   toast/snackbar mechanism in the app — a fixed-position `#toast` element outside the overlay system
   so it survives every overlay closing at once) confirms "Search record saved" / "New site
   registered", since otherwise the screen just flicks back to idle with no feedback. There's no
   persistent "session" wrapper like goanna's
   hunting session: each site visit is one standalone submission, matching the Fulcrum data model
   this app replaces (see "Fulcrum migration" below). Backing out of the site sheet without confirming
   (backdrop tap) re-opens the survey-results sheet rather than discarding the search — its field
   values are still intact since nothing clears them until a site is actually confirmed.

   Separately, `#registerSiteBtn` on the main screen opens the same site sheet in
   `openSiteSheet('registerOnly')` mode (forced to "new site", existing-site tab hidden) for the
   day-trip site-registration shortcut described above — confirming there submits immediately with
   no timer/survey step at all.

   **Existing-site picker: map-first, list as fallback.** Site names are often absent or ambiguous
   ("Dam", a raw coordinate pair), so picking from a text list doesn't help a crew standing at a
   physical waterpoint recognize whether it's already known — a map does. `#existingSitePanel`'s
   primary action is `#viewMapBtn` ("View on map"), opening a dedicated full-screen overlay
   (`#mapOverlay`, `openMapPicker()`) rather than embedding the map inside the bottom sheet — `.sheet`
   is `overflow-y: auto`, and a draggable/pinch-zoomable map inside a container that also wants to
   capture vertical scroll is a real gesture-conflict risk, so this follows the same full-screen
   pattern already used for the QR camera view (`#scanOverlay`) instead. The original search+list UI
   still exists, demoted behind an "or choose from a list" toggle (`#listPickerPanel`) as a safety net
   if a device's GPS or map rendering misbehaves in the field — nothing about it changed except that
   `renderSiteList()` now sorts by distance from the crew's current position (nearest first) and shows
   that distance per row when a position is available, falling back to today's unsorted/no-distance
   rendering otherwise (`haversineMeters()`/`formatDistance()`).

   The map is vendored **Leaflet** (`L`), pasted inline the same way jsQR is (see "Notes for editing"
   below) — chosen over hand-rolling pan/zoom/projection math because there's no test suite in this
   repo to catch subtle bugs in that kind of code, and Leaflet ships a proper standalone
   `leaflet.js`/`leaflet.css` distributable meant for exactly this kind of inclusion, so vendoring it
   doesn't require adopting a build step. Tiles are OpenStreetMap's standard free tile servers
   (`tile.openstreetmap.org`) — no signup, no API key, acceptable under OSM's usage policy for a small
   field-crew app's request volume. Markers use `L.divIcon()` (small CSS circles, on-brand colors)
   rather than Leaflet's default pin images: Leaflet infers its default icon path from its own
   `<script src>`, which doesn't exist when the library has no `src` at all (pasted inline) — divIcon
   sidesteps that gotcha entirely and needs no bundled marker image assets. **Online/offline is simply
   "is the tile layer attached or not"** (`setMapTileLayer()`, hooked into the app's existing
   `window.addEventListener('online'/'offline', ...)`), not two separate code paths — Leaflet's
   coordinate/pan/zoom system doesn't depend on tiles being present, so markers, pan, and zoom all
   work identically with or without them; offline just shows a caption instead of the basemap. A
   `currentPosition` GPS fix is captured once per `finalize` site-selection step
   (`captureCurrentPositionForSiteSelection()`) and shared by both the map's "you are here" marker and
   the list's distance sort, rather than each requesting location separately.

   The picker always opens at a **~300 m radius** view (`setInitialMapView()` →
   `fitBounds(L.latLng(...).toBounds(600), ...)`), centred on the crew's fix, or on the fixed West
   Kimberley fallback point (`[-18.0, 124.0]`) when there's no fix yet — deliberately *not* an
   all-sites overview, which across this landscape is a near-continental zoom. If the fix lands after
   the map is already open (the sheet's GPS request races the "View on map" tap),
   `mapPendingInitialRecenter` lets `captureCurrentPositionForSiteSelection()`'s callback re-centre it
   once — cancelled if the crew pans first (`dragstart`).

   Tile image requests aren't special-cased in `sw.js` — its fetch handler already intercepts every
   GET request with no origin filter (see "PWA shell" below), so OSM tiles get opportunistically
   cached alongside the app shell for free, meaning a recently-viewed area stays visible offline next
   time. There's no eviction, so this cache can grow unbounded over a long deployment — a known,
   deliberately deferred tradeoff, not an oversight; revisit only if storage becomes an actual problem
   in the field.

   **Proximity warning on new-site registration.** Both new-site paths (the day-trip
   `registerOnly` shortcut and the `finalize` flow's "Register new site" tab) share the same
   `confirmSiteBtn` handler branch, so a single check there covers both: right after `currentSite`
   is built and before `finishSiteConfirmation()` would run, `nearbyKnownSites(lat, lon,
   NEARBY_SITE_WARNING_M)` (300m, a named constant next to `haversineMeters`/`formatDistance`) checks
   the about-to-be-submitted coordinates against every known site. If any are within range,
   `finishSiteConfirmation()` is *not* called yet — instead `openMapPicker()` opens in a "warning"
   mode (`mapWarningActive`) that reuses the ordinary map picker's rendering (tiles, markers, popups)
   but swaps in a warning banner, a `.newSiteMarker`-styled pin at the proposed coordinates (distinct
   from the GPS "you are here" marker, since these coordinates may have been hand-edited away from
   the live GPS fix), a tight `fitBounds` around just the proposed point and the nearby site(s)
   instead of the ordinary picker's ~300 m current-location view, and two explicit actions in place
   of the normal flow
   (`#mapWarningCancelBtn`/`#mapWarningContinueBtn`) — "Cancel" just closes the overlay with nothing
   saved, "Register anyway" calls the same `finishSiteConfirmation()` that would have run without the
   warning. Existing-site popups' "Use this site" button is suppressed while `mapWarningActive` is
   true: picking the nearby site from here would produce a pointless record
   (`survey_conducted:'no'` + `is_new_site:'no'`, i.e. no new entity and no real survey data) — this
   mode is deliberately informational-only, not an alternate picker.
5. **Record lifecycle** — same `pending → synced`/`failed` shape as the goanna app, one record per
   site visit (no multi-record "session" to finish). Field names on the record object are kept
   **identical to the XLSForm survey sheet's `name` column** (no separate `FIELD_MAP` translation
   layer like the goanna app has) — deliberately, to remove one source of drift between the two
   files; if you rename a field in one, rename it in the other. `survey_conducted` ('yes' on a
   normal timed-search record, 'no' on a registration-only one) follows this same rule.
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

Verified against a **live Central 2026.x** by the Fulcrum migration (see `FULCRUM_MIGRATION.md`;
scripts in `../invasion-front-monitoring/src/fulcrum-migration/`): posting
`baseVersion=""`/`trunkVersion=""`/`branchId=""` on a plain `update="false"` reference to an
existing entity *is* accepted — 78 of 80 historical submissions built in exactly
`buildSubmissionXml()`'s shape (`create="false" update="false"`, referencing a real entity `id`)
were accepted by Central with `200`; the other 2 were `409` duplicate-`instanceID` conflicts from a
re-run, not shape/validation rejections. That resolves the "update an existing entity" case. It
does **not** by itself resolve entity *creation*: the migration authenticated as a **web user** and
posted to Central's REST endpoint (`POST /v1/projects/:id/forms/:xmlFormId/submissions`), not this
app's app-user-scoped OpenRosa `xml_submission_file` path, and it created entities via Central's
bulk "Upload CSV" dataset page (an admin/web-user action), never via a submission's `create="true"`
— see below for where the app's own create path was verified instead.

One more thing the migration surfaced: Central's **entity-list CSV download** (from a dataset's
page in the Central UI) is a *different* shape from the CSV Central auto-attaches to a form's
manifest for `select_one_from_file` — it uses `__id` (plus `__createdAt`/`__creatorId`/
`__updates`/etc.) as the identity/system columns, whereas the manifest-attached CSV this app's own
`siteFromCsvRow()` parses uses `name` (see `toad-monitoring-app.html`'s `siteFromCsvRow()`, and the
`id_col_candidates` comment in `apply-entity-id-crosswalk.R`). Don't assume a script or doc talking
about one CSV's columns applies to the other.

Verified against a live Central 2026.x via an actual **app-user session** in the app itself (not a
migration script): Central *does* auto-attach `toad_monitoring_sites.csv` to this form's manifest
for app users — `fetchSiteList()`'s manifest fetch found and downloaded it correctly once one bug
was fixed (see below).

That bug, now fixed: `fetchSiteList()`'s two `fetch()` calls (the manifest GET and the CSV
download GET) sent no headers at all, so Central rejected the manifest request with `400`. Central
requires `X-OpenRosa-Version: 1.0` on *every* OpenRosa request — formList/manifest/submission
alike, GET or POST — "or the request will be rejected" (per
[docs.getodk.org's OpenRosa Endpoints reference](https://docs.getodk.org/central-api-openrosa-endpoints/)).
`submitOne()`'s submission POST already set this header; `fetchSiteList()`'s GETs now do too. If a
future OpenRosa fetch is added to this app, give it the same header from the start rather than
rediscovering this via a live 400.

Verified against a live Central 2026.x via an actual **app-user session** in the app itself: the
new-site registration path (`is_new_site='yes'`) — a bare OpenRosa `xml_submission_file` POST with
`create="true"` in the `<meta><entity>` block, no ODK Collect and no web-user REST call involved —
is accepted, creates the entity, and both the entity's properties and the survey's own answers land
in Central in the right places. This was the last unverified piece of the entities design; nothing
about entity creation or the OpenRosa submission path remains unexercised against a live Central.

That verification was of a `create="true"` submission carrying real survey answers (the normal
end-of-visit flow). The day-trip "Register a new site" shortcut (`survey_conducted='no'`) posts the
same shape — same entity block, same `create="true"` — just with the five survey-outcome fields
blank instead of filled in. Structurally identical, so this should behave the same, but that
specific blank-fields case hasn't itself been separately exercised against a live Central yet —
worth a quick check before relying on it in the field.

### PWA shell (`manifest.json`, `sw.js`, `index.html`)

Identical mechanism to the goanna app — see its CLAUDE.md. `CACHE_NAME` here is
`tcz-toad-shell-v6`; bump it whenever you change what needs to be cached.

### Styling

Same CSS-custom-properties approach as the goanna app, deliberately re-themed rather than reusing
its exact palette so the two apps are visually distinguishable at a glance in the field (crews may
have both installed): swamp-green/clay/cream (`--marsh`, `--clay`, `--rock`, `--cream`) instead of
goanna's olive/dirt. Same single-column mobile shell, same bottom-sheet modal pattern.

## Removed: native title area & property

The app used to capture two per-site fields — **native title area** (`native_title_area`, mnemonic
`nta_id` values like `NTA-KJ`) and **pastoral property** (`property`, mnemonic `property_id` values
like `PROP-YM`) — as `<select>`s on the new-site form (`#newSiteNta` / `#newSiteProperty`), backed
by `nta_choices` / `property_choices` in `toad_detection_survey.xlsx` and hand-copied from
`../shared-taxonomy/taxonomy/native_title_areas.csv` / `properties.csv`.

They were dropped once the full pipeline (app → ODK Central → ETL → reporting app) was in place:
the `data-warehousing` ETL now derives both from each record's geolocation via a spatial join
against NTA / station boundaries, so collecting them in the field was redundant. Removed
end-to-end — the two `survey` rows and both choice lists in the XLSForm, the `<select>`s and
`NTA_LABELS` / `PROPERTY_LABELS` maps and `siteBadges()` entries in the HTML, the
`<native_title_area>` / `<property>` elements in `buildSubmissionXml()`, and the CSV columns
`siteFromCsvRow()` used to read. The `toad_monitoring_sites` entity dataset in Central keeps its
existing `native_title_area` / `property` property columns (Central never deletes entity
properties) with their historical values; new submissions simply don't write them and the app no
longer reads them. `../shared-taxonomy` remains authoritative for the ETL's boundary/reference
data — the app just no longer carries a snapshot of it.

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
- Leaflet (`leaflet.js`/`leaflet.css`, ~160KB total) is vendored inline the same way — pasted
  as-is from the library's own pre-built distributable, not hand-modified. If it ever needs
  upgrading, re-fetch both files for the new version and paste over the existing blocks rather than
  patching in place; don't add a build step or CDN reference to avoid re-vendoring by hand.
- `crypto.randomUUID()` generates both `site_id` (which doubles as the entity's real key when
  `is_new_site='yes'`) and the OpenRosa `instanceID` — don't change ID generation without
  preserving uniqueness guarantees, and don't conflate the two IDs even though the same function
  produces both.
