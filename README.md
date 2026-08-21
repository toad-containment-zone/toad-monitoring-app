# Cane Toad Time-to-Detection Survey — TCZ

A field app for the Toad Containment Zone (TCZ) program: times how long it takes to detect a
cane toad at a waterpoint site, then syncs the records to our ODK Central form. Sites are
registered once and reused across years (they're stored as ODK Central **Entities**), so repeat
visits just pick the site from a list instead of re-entering its details. Works fully offline once
installed — no signal needed out in the field.

**App URL:** [https://toad-containment-zone.github.io/toad-monitoring-app/toad-monitoring-app.html]

## Setup (do this once)

You'll need an app user QR code or text code from whoever manages the TCZ program's ODK Central
project before you start — see step 2 below.

### 1. Install the app on your phone

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

### 2. Add your app user code

1. Open the app and tap the ⚙ settings icon.
2. Under **Central app user code**, either tap **📷 Scan QR code instead** and scan the code you
   were sent, or paste the text code directly.
3. Tap **Save settings**. If you're online, the app will also fetch the current list of known
   sites at this point.

If your phone asks for camera or location permission the first time you use the QR scanner or
register a new site, allow it — the camera is only used to read the QR code, and location is only
used to prefill a new site's coordinates (you can always edit them by hand). If location isn't
working, the app will pop up a message explaining why (e.g. permission is off, or GPS/location
services are turned off on the phone) the next time you open it — on Android this is sometimes a
separate permission from the one Chrome itself asks for, under your phone's **Settings → Apps →
Chrome → Permissions → Location**. Either way, you can always type a new site's coordinates in by
hand instead of relying on GPS.

## Using it in the field

- The timer is the first thing you see when you open the app — tap **Start search** as soon as you
  arrive at a waterpoint, then **Stop search** once the search ends (whether or not a toad was
  found).
- Fill in the quick follow-up details (people searching, temperature, water access, whether a toad
  was found, notes) and tap **Next: choose site**.
- Say which site this search was at:
  - **Existing site** — tap **🗺️ View on map** to find it visually (your current location is shown
    too, and known sites are marked), or use **or choose from a list** to search by name instead if
    GPS or the map isn't cooperating.
  - **Register new site** if this waterpoint hasn't been surveyed before — fill in its location
    (GPS-prefilled, but you can correct it by hand), name, waterpoint type, native title area, and
    property once; you won't need to re-enter them on future visits. If the coordinates are close to
    a site that's already known, the app shows a map warning first so you can check it isn't a
    duplicate before saving.
- Setting up new sites for a future visit without timing a search there yet? Tap **Register a new
  site** on the main screen — it registers the site straight away with no timer or survey step
  involved.
- Records sync automatically whenever you have signal (or tap **Sync now**). Nothing is lost if
  you're offline — everything queues on the phone until it can sync, including newly-registered
  sites (shown as "pending sync" in the site list until they go through).

## Historical data

Several years of time-to-detection data were previously collected in Fulcrum. See
[`FULCRUM_MIGRATION.md`](FULCRUM_MIGRATION.md) for the field mapping and import plan for bringing
that history into ODK Central alongside data collected with this app.

## For developers

See [`CLAUDE.md`](CLAUDE.md) for how the app is built and how to make changes.
