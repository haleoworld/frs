# Deploying Ember

A reference for hosting the app, syncing data, and installing it on every device. After first-time setup, daily use is just tapping the home-screen icon — no maintenance.

## Part 1 — Hosting (GitHub Pages)

The app is hosted on **GitHub Pages** from the repo `haleoworld/frs`. There's nothing to drag or upload manually:

1. **Push the file to the repo.** Any change to `index.html` that lands on the Pages-serving branch (e.g. `main`) is published automatically.
2. **The URL stays constant.** GitHub Pages serves it at `https://haleoworld.github.io/frs/` (or your custom domain if configured). Every device picks up the new version on the next page load.

That's the whole deploy step: **push, and it's live.** No build, no servers, no SSL to manage.

> If you ever need to confirm or change Pages settings: GitHub repo → **Settings → Pages** → "Build and deployment" → Source = your branch, folder = `/ (root)`.

## Part 2 — Cloud sync (GitHub Gist)

Data syncs across devices through a **private GitHub Gist** you own. Every save becomes a versioned revision, so you can roll back to an earlier copy if anything ever goes wrong.

### First device (one time, ~2 minutes)

1. Create a token: go to **github.com/settings/tokens/new**, name it "Ember sync", and under **scopes tick only `gist`** — nothing else. Choose an expiry (or "No expiration"), then **Generate token** and copy it (starts with `ghp_`).
2. Open the app → **Settings → Cloud sync**. Paste the token, **leave Gist ID blank**, and click **Save & test**.
3. The app creates a new private gist, saves your data, and shows the **Gist ID** just above the button. Copy that ID — you'll need it on other devices.

### Every other device

1. Open the app → **Settings → Cloud sync**.
2. Paste the **same token** and the **Gist ID** from the first device.
3. Click **Save & test**, then **Pull from cloud**. Your data appears.

> Your token lives only in that browser's local storage and is **never written into the gist** — the app strips it out before uploading.

## Part 3 — Install on each device (~30 seconds per device)

"Add to Home Screen" makes the app open like a native app (no browser chrome, app-icon launch).

### iPhone / iPad (Safari)

1. Open the app URL in **Safari** (only Safari can install web apps on iOS).
2. Tap the Share button → scroll down → **Add to Home Screen** → **Add**.
3. Tap the new icon (purple radar) to open it like a native app.
4. Set up sync once (Part 2) if this is a new device.

### Android (Chrome)

1. Open the URL in **Chrome**.
2. ⋮ menu → **Install app** / **Add to Home screen** → confirm.
3. Set up sync once (Part 2) if needed.

### Laptop / Desktop (Chrome, Edge, or Safari)

- **Chrome/Edge:** click the install icon in the address bar, or ⋮ menu → Install.
- **Safari (macOS):** File menu → **Add to Dock**.
- **Any browser:** bookmarking the URL also works fine.

## Part 4 — Updating the app later

When `index.html` changes (new feature, bug fix, tweak):

1. **Push the new file to the repo.** GitHub Pages redeploys automatically; the URL is unchanged.
2. Each device picks up the update on next open. If not, pull-to-refresh (mobile) or Cmd/Ctrl+R (desktop).

Data is unaffected by updates — it lives separately in localStorage and your gist.

## Restoring an older copy of your data

Because every save is a gist revision, you have a full history:

1. Open your gist on **gist.github.com** (Settings → Cloud sync → "open on GitHub").
2. Click **Revisions**, pick an earlier save, and view its raw `family-rewards-data.json`.
3. Copy that JSON and paste it into the app via **Settings → Import JSON**.

## Quick troubleshooting

- **"My data isn't showing on a new device"** — Settings → Cloud sync → paste the same **token + Gist ID** → Save & test → Pull from cloud. Each browser has its own local storage; the gist is the source of truth.
- **"Sync failed / bad token"** — the token may be wrong, expired, or missing the `gist` scope. Generate a fresh one (Part 2, step 1) and paste it again.
- **"It looks like a regular browser page, not an app"** — open it via the home-screen icon you added, not a bookmark. Standalone mode only kicks in for the installed shortcut.
- **"The page didn't update after I pushed"** — give GitHub Pages a minute, then hard-refresh (Cmd/Ctrl+R) or pull-to-refresh.

## Cost & maintenance summary

- **GitHub Pages:** $0 forever for this workload.
- **GitHub Gist:** $0 forever; uses negligible quota of your GitHub account.
- **Maintenance:** effectively zero. No certs, servers, or DNS to manage. The only thing that can expire is your sync token (if you set an expiry) — regenerate it and re-paste.
