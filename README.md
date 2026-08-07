# Cane Toad Time-to-Detection Survey — TCZ

A field app for the Toad Containment Zone (TCZ) program: times how long it takes to detect a
cane toad at a waterpoint site, then syncs the records to our ODK Central form. Sites are
registered once and reused across years (they're stored as ODK Central **Entities**), so repeat
visits just pick the site from a list instead of re-entering its details. Works fully offline once
installed — no signal needed out in the field.

**App URL:** [https://toad-containment-zone.github.io/toad-monitoring-app/toad-monitoring-app.html]

## Setup (do this once)

### 1. Admin: set up the ODK Central project

Before any field device can use this app, someone with admin access to the ODK Central project
needs to:

1. Upload `toad_detection_survey.xlsx` as a form in the project. Publishing it will create the
   `toad_monitoring_sites` entity dataset (declared on the workbook's `entities` sheet).
2. Create an **app user** for the field crew/device and grant it access to the
   `toad_detection_survey` form.
3. Confirm the app user can download the form's `manifest` and its `toad_monitoring_sites.csv`
   attachment — this app relies on that (the same offline-entity-list mechanism ODK Collect uses)
   to show field crews the list of already-registered sites. If sites aren't showing up on a
   device that should have connectivity, this is the first thing to check in Central's project
   settings.
4. Send the app user's QR code / long text code to the field crew (from the app user's page in
   Central — "QR code" / "Configure a client with this code").

### 2. Install the app on your phone

**iOS (must use Safari — other browsers don't support this):**
1. Open the app URL above in Safari.
2. Tap the **Share** button (square with an arrow pointing up).
3. Scroll down and tap **Add to Home Screen**, then tap **Add**.

**Android (Chrome):**
1. Open the app URL above in Chrome.
2. Tap the **⋮** menu (top right).
3. Tap **Add to Home screen** (or **Install app**, if Chrome offers it directly), then confirm.

Either way you'll get an app icon on your home screen. Open it once while you have signal so it
can finish setting itself up for offline use — after that it'll launch instantly with zero bars.

### 3. Add your app user code

1. Open the app and tap the ⚙ settings icon.
2. Under **Central app user code**, either tap **📷 Scan QR code instead** and scan the code you
   were sent, or paste the text code directly.
3. Tap **Save settings**. If you're online, the app will also fetch the current list of known
   sites at this point.

If your phone asks for camera or location permission the first time you use the QR scanner or
register a new site, allow it — the camera is only used to read the QR code, and location is only
used to prefill a new site's coordinates (you can always edit them by hand).

## Using it in the field

- Tap **Select site** to begin. Choose an existing site from the list (search by name), or
  **Register new site** if this waterpoint hasn't been surveyed before — fill in its location,
  waterpoint type, and native title area once; you won't need to re-enter them on future visits.
- Use the timer to record the search: **Start search**, then **Stop search** once the search ends
  (whether or not a toad was found).
- Fill in the quick follow-up details (people searching, temperature, water access, whether a
  toad was found, notes) and tap **Save record**.
- Records sync automatically whenever you have signal (or tap **Sync now**). Nothing is lost if
  you're offline — everything queues on the phone until it can sync, including newly-registered
  sites (shown as "pending sync" in the site list until they go through).
- Tap **Change site** at any point to switch to a different site.

## Historical data

Several years of time-to-detection data were previously collected in Fulcrum. See
[`FULCRUM_MIGRATION.md`](FULCRUM_MIGRATION.md) for the field mapping and import plan for bringing
that history into ODK Central alongside data collected with this app.

## For developers

See [`CLAUDE.md`](CLAUDE.md) for how the app is built and how to make changes.
