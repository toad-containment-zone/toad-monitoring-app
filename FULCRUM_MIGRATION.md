# Migrating historical Fulcrum data into ODK Central

Several years of cane toad time-to-detection data were collected in Fulcrum before this app
existed. This document is a **plan and field mapping**, not a working import script — no Fulcrum
export was available while drafting this app, so the exact column names/types in your export may
differ slightly from what's assumed below. Treat this as a checklist to adapt once you have a real
export in hand, not something to run unmodified.

## Why this isn't a single mechanical step

Fulcrum has no equivalent of an ODK Entity: every visit to a site is presumably a flat record with
that visit's site fields (lat/lon, waterpoint type, etc.) repeated on it, not a reference to a
persistent site object. Importing into this app's schema means **first reconstructing which
historical records refer to the same physical site**, so that:

- each unique site becomes exactly one `toad_monitoring_sites` entity (not one entity per visit),
- every historical visit record references that entity's ID instead of repeating its properties.

That reconstruction (grouping Fulcrum records into sites) is inherently fuzzy — matching on site
name where present, and on GPS proximity (e.g. within ~50–100m, adjusted for your sites' actual
spacing) otherwise — and worth a manual review pass before treating it as final, rather than fully
trusting an automated clustering. Get this step wrong and you'll either merge two real sites into
one, or split one real site into several duplicate entities.

## Field mapping

### Site-level (Fulcrum) → `toad_monitoring_sites` entity properties

| Fulcrum field       | Entity property        | Notes |
|----------------------|-------------------------|-------|
| latitude              | `latitude`              | decimal |
| longitude             | `longitude`             | decimal |
| site name (optional)  | `site_name`             | also feeds the entity `label` if present, else falls back to `"lat, lon"` — see `buildSubmissionXml()` in `toad-monitoring-app.html` |
| type of waterpoint (multi-choice) | `waterpoint_type` | space-separated slugs — see value mapping below |
| type of waterpoint: "other" free text (if Fulcrum captured it) | `waterpoint_type_other` | |
| natural or artificial  | `water_origin`          | `natural` / `artificial` |
| native title area      | `native_title_area`     | see NTA mapping below — this field likely doesn't exist in Fulcrum at all and will need to be back-filled by cross-referencing site locations against `../shared-taxonomy/taxonomy/native_title_areas.csv` / `property_nta_overlap.csv`, not copied from an existing column |
| property (pastoral station) | `property`       | stores the `property_id` mnemonic (e.g. `PROP-YM`), not a free-text name — see property mapping below. Like NTA, this likely doesn't exist in Fulcrum as a clean column and needs back-filling from `../shared-taxonomy/taxonomy/properties.csv`, cross-referenced against site location |

**Waterpoint type value mapping** (Fulcrum label → this app's slug):

| Fulcrum choice | Slug |
|---|---|
| Creek | `creek` |
| Dam/turkey nest/impoundment | `dam_turkey_nest_impoundment` |
| Roadside scrape | `roadside_scrape` |
| Tank and trough | `tank_and_trough` |
| Outlet pipe | `outlet_pipe` |
| Waterhole | `waterhole` |
| Soak or spring | `soak_or_spring` |
| other | `other` |

**Property value mapping** (station name → `property_id`, per `../shared-taxonomy/taxonomy/properties.csv`):

| Station name | `property_id` |
|---|---|
| Frazier Downs Station | `PROP-FDS` |
| Shamrock Station | `PROP-SHAMS` |
| Anna Plains Station | `PROP-APS` |
| Nita Downs Station | `PROP-NDS` |
| Shamrock Gardens | `PROP-SHAMG` |
| Shelamar Station | `PROP-SHEL` |
| Roebuck Plains Station | `PROP-RPS` |
| Dampier Downs | `PROP-DD` |
| Liveringa | `PROP-LIV` |
| Myroodah | `PROP-MYR` |
| Yakka Munga | `PROP-YM` |
| Yeeda | `PROP-YEE` |
| *(unknown/not on a mapped property)* | `none` |

The last five rows plus the Roebuck Plains alias were added to `properties.csv` specifically to
cover this migration's source data — that catalog previously only had stations further north/east
(Karajarri/Nyangumarta country) and didn't include these Fitzroy Valley stations at all. If
`sites_for_central_upload.csv`-style exports use plain station names (as this migration's did)
rather than `property_id` codes, map them through this table (and extend `properties.csv` first,
via PR, for any station not yet listed) before uploading — don't invent new codes ad hoc per
import.

### Survey-level (Fulcrum) → `toad_detection_survey` submission fields

| Fulcrum field | Submission field | Notes |
|---|---|---|
| date | `visit_date` | ISO `YYYY-MM-DD` |
| time | `visit_time` | 24h `HH:MM` |
| temperature | `temperature_c` | confirm Fulcrum's units — this app assumes °C |
| can toads access water? | `toads_access_water` | `yes` / `no` |
| how many people | `people_searching` | integer |
| time searched (minutes) | `time_searched_minutes` | integer — this app's live timer rounds to the nearest minute, so historical precision should already match |
| toad found, or not | `toad_found` | `yes` / `no` |
| notes | `notes` | free text |
| *(n/a)* | `is_new_site` | `yes` only on the one imported record chosen to "own" each site's entity creation (see below); `no` on every other visit to that same site |
| *(n/a)* | `site_id` | the entity's assigned ID (see below) — same value on every record for that site |

## Suggested import procedure

1. **Export** Fulcrum data for this app to CSV (or GeoJSON, then flatten to CSV).
2. **Cluster into sites** per the site-name/GPS-proximity approach above; assign a temporary
   cluster ID to every row. Review the clustering manually — this is the step most worth a human
   look before proceeding.
3. **Create the entities** — the cleanest path is Central's **bulk CSV upload** for a dataset
   (from the dataset's page in the Central UI: Upload CSV), giving each cluster's first/most
   complete record's site fields as one row, with the entity `label` set to `site_name` (or
   `"lat, lon"` if blank) to match this app's convention. This is an *admin* action using a
   Central web-user session, not the field app's app-user-scoped OpenRosa path — appropriate here
   since it's a one-off bulk load, not routine field submission.

   **Confirmed against a live Central 2026.x**: this upload only accepts `label` plus one column
   per declared entity property, headers matching exactly (including order) the property order
   `toad_detection_survey.xlsx`'s survey sheet declares via `save_to` — no ID column. Central
   assigns each entity's own UUID at creation; there is no way to supply your own through this
   path. That means whatever placeholder site UUID you used to build step 4's submissions (before
   any entity existed to reference) is *not* the entity's real ID, and has to be reconciled
   afterwards: download the resulting entity list from that dataset's page in Central's UI once
   the upload's done, and match it back to your placeholder IDs by **coordinates**, not label —
   this migration's `label`s aren't reliably unique (site-name clustering was dropped, see the
   note above the `PROXIMITY_THRESHOLD_M` constant, so two genuinely different sites can share a
   name like "Dam") — see
   `invasion-front-monitoring/src/fulcrum-migration/apply-entity-id-crosswalk.R`, which does
   exactly this (exact-coordinate match first, falling back to nearest-within-150m, refusing to
   guess on anything ambiguous) and rewrites the affected submission XML in place — run it before
   step 4's POSTs go out, not after.
4. **Create the historical submissions** — for each Fulcrum row, build a submission XML using the
   same shape as `buildSubmissionXml()` in `toad-monitoring-app.html`, with:
   - `is_new_site` = `no` for every row (the entities already exist from step 3 — don't recreate
     them from historical visit records too),
   - `site_id` = that row's cluster's entity's **real** ID — the placeholder UUID rewritten to
     Central's assigned ID per step 3's crosswalk, not the placeholder itself,
   - the survey fields per the mapping table above,
   - a freshly generated `instanceID` (these are historical records being imported now, not
     literally resubmitted from the original device).
   POST each one to the form's submission endpoint — note the URL is keyed on the form's
   *xmlFormId*, `toad_detection_survey`, not its display title "Cane Toad Detection Survey".
   `invasion-front-monitoring/src/fulcrum-migration/upload-submissions.R` does this: logs in as a
   web user (`POST /v1/sessions`) and POSTs each file's raw XML to `POST
   /v1/projects/:id/forms/toad_detection_survey/submissions`. Test it against 1-2 records first
   (its `CENTRAL_UPLOAD_LIMIT` env var) and confirm they land against the right entity in Central's
   UI before uploading the full batch. **Update**: this raw-XML REST path has since been run
   against a live Central 2026.x for the actual migration — 78 of 80 historical submissions built
   in this shape (`create="false" update="false"`, referencing a real entity `id`) were accepted
   with `200`; the other 2 were `409` duplicate-`instanceID` conflicts from a re-run, not
   shape/validation rejections. See `toad-monitoring-app.html`'s CLAUDE.md, "Entities: what's
   verified vs. assumed", for the full picture — including that the app's own field-facing
   new-site-creation path (`is_new_site='yes'`, `create="true"`, posted via the app-user OpenRosa
   `xml_submission_file` endpoint rather than this migration's web-user REST/bulk-CSV paths) has
   now separately been verified live too, end-to-end through the app itself.
5. **Spot-check** a handful of imported sites in the app itself (or via Central's UI) — confirm a
   site that should have multiple historical visits shows up once as an entity with several
   submissions against it, not several near-duplicate entities.

## Open questions to resolve against the real export

- Whether Fulcrum's "type of waterpoint" and "natural or artificial" were ever left blank, and
  what to do with a historical site that's missing required-for-new-site fields (this app's XLSForm
  marks few site fields as `required`, so blanks should still be *accepted* — but decide whether
  they should be back-filled instead of imported blank).
- Whether Fulcrum recorded anything usable for `native_title_area` (a governance-relevant field
  that predates this app and Fulcrum both) or whether it must be entirely back-filled from
  `../shared-taxonomy`.
- Whether Fulcrum recorded a usable `property` (pastoral station) field, or whether it must be
  entirely back-filled from a spatial join against station boundaries, same as `native_title_area`
  — and whether every station touched by the real export is already in
  `../shared-taxonomy/taxonomy/properties.csv` (extend it via PR first if not, per the property
  mapping table above).
- Whether temperature was recorded in °C consistently.
