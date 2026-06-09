# Family Reward System — Project Context

A single-file HTML reward tracker for Jerry's family. Self-contained, runs in any browser, no server, no LLM calls at runtime. **Cost to operate: $0.**

## File map

- `family-rewards.html` — the entire app (HTML + CSS + JS in one file, ~3100 lines after sync + PWA + rewards + comments additions)
- `CLAUDE.md` — this file (project context for future Claude sessions)
- `DEPLOY.md` — Jerry-facing deployment + install instructions (GitHub Pages + per-platform Add to Home Screen). Refer Jerry here when he asks about hosting, updating, or installing on a new device.
- No other source files. Nothing to install or build.

## Goal & user

Jerry wants to log kids' incremental activities (housework, homework, sharing, brushing teeth, getting ready, etc.) from phone or laptop and see weighted points roll up into charts by day/week/month and total by person. Starts with two placeholder kids; family may grow.

## Hosting & cross-platform access

**Hosting: GitHub Pages** (switched from Netlify on/around 2026-05-26). Jerry pushes the file to his GitHub repo (`haleoworld/frs`); GitHub Pages serves it automatically on push. Free forever, https, zero maintenance. Updates deploy by pushing the new file — no manual drag step.

**Add to Home Screen.** The HTML includes a PWA manifest (inlined as a base64 data URI), apple-touch-icon, and `apple-mobile-web-app-capable` meta tags so the app installs as a standalone home-screen icon on iOS Safari, Android Chrome, and desktop Chrome/Edge/Safari. Icon is a purple radar/attribute SVG (concentric arcs + an arrow to the center) used for the favicon, apple-touch-icon, and manifest icon.

**Important consequence: localStorage is per origin.** When Jerry opens the app on a new device/browser, it sees fresh localStorage. He pastes the GitHub token + Gist ID once on each device (Settings → Cloud sync), then Pull from cloud restores all data. This is documented in DEPLOY.md.

## Cross-device sync model

**Goal:** automatic cross-device sync — settings and data save and reload across phone and desktop without the user touching a JSON file.

**Backend: GitHub Gist** (migrated from Google Sheets / Apps Script on 2026-05-26). Every `save()` PATCHes the entire `state` JSON into one file (`family-rewards-data.json`) in a private gist the user owns. Chosen because **GitHub versions every PATCH**, so the gist's Revisions tab is a free, restorable backup history — the reason Jerry wanted the switch (roll back if a sync ever corrupts the data). Free, data stays in Jerry's GitHub account, works in any browser. Auth is a **classic Personal Access Token scoped to only `gist`**, stored in `localStorage` per device. Setup instructions live in the in-app Settings → Sync card under a collapsible section.

Always-present fallback: localStorage per device + manual Export/Import JSON (Settings → Data). Leaving the token blank = offline mode.

**Key implementation details:**

- `state.settings.gistToken` — the GitHub PAT (gist scope). Empty string = no sync (offline mode).
- `state.settings.gistId` — the gist holding the data file. Blank on the first device → the first push POSTs a new private gist and stores the returned id. Other devices paste token + this id.
- `state.settings.lastSyncedAt` — ISO timestamp of last successful push or pull.
- `GIST_API` = `https://api.github.com/gists`; `GIST_FILENAME` = `family-rewards-data.json`. `gistHeaders(token)` builds the Bearer auth + API-version headers. `extractGistId(s)` accepts a raw id or a full gist URL.
- `syncPayload()` — serializes `state` **with `gistToken` and `gistId` blanked out**, so credentials are NEVER written into the gist (secret gists are readable by anyone with the link). MUST be used for every upload body.
- `pushToCloud()` — PATCHes the gist if `gistId` is set, otherwise POSTs to create one and captures `data.id`. Body always comes from `syncPayload()`.
- `pullFromCloud()` — GETs the gist, parses the data file, `migrate()`s it, replaces local state if the remote `settings.lastSaved` is newer (last-write-wins). **Preserves this device's own `gistToken`/`gistId`** after adopting remote (the cloud copy has them blanked).
- `queuePush()` — debounces pushes 1.5s after the last `save()` so a flurry of logs becomes one request.
- On boot, if both `gistToken` and `gistId` are set, pull silently. Same on window focus.

**Save & test handler — DO NOT call `save()` before reading the gist.**

This caused a real data-loss bug under the old backend on 2026-05-17, and the same hazard exists with Gist. On a fresh device the user pastes token + gist id and clicks Save & test. If the handler does `save()` first, `lastSaved` bumps to "now" → the merge logic thinks local is newer → it overwrites good cloud data, and the debounced `queuePush()` fires a second time.

The correct flow (currently implemented):
1. If a gist id is supplied, GET the gist directly with `fetch()` — do NOT mutate or save state yet.
2. Inspect both sides: `cloudHasData` = remote has records, `localHasData` = local has records.
3. Both have data → `confirm()` asks which side wins; adopt-cloud uses `saveLocal()` (no push), keep-local uses `save()` + forced push.
4. Only cloud has data (typical new-device case) → adopt cloud via `saveLocal()`.
5. Cloud empty/unreadable but a gist id was given → push local up.
6. **No gist id supplied** → it's safe to `save()` then push, because there's no existing cloud copy to overwrite; the push auto-creates the gist and stores its id.

Credentials (`gistToken`/`gistId`) are set as part of the chosen action, not before it. `saveLocal()` persists without triggering a push (used after pulling).

**Design constraints (still apply for any future sync change):** keep `save()`/`load()` as the only mutation points; debounce remote writes; last-write-wins on conflicts (no auto-merge — it's <10 records/day); never store the token anywhere it could leak (it lives only in localStorage, and `syncPayload()` strips it from uploads).

## Data model (single object in `localStorage['familyRewards.v1']` — key name preserved across schema bumps)

```js
{
  version: 7,
  settings: { theme, activePersonId, lastSaved, lastSyncedAt, gistId, gistToken },
  people:                [{ id, name, color, emoji }],
  categories:            [{ id, name, weight, color }],              // activity categories
  activities:            [{ id, name, categoryId, basePoints, emoji }],
  rewardCategories:      [{ id, name, weight, color }],              // multiplier on reward costs
  rewards:               [{ id, name, cost, categoryId, emoji, color }],
  misbehaviorCategories: [{ id, name, weight, color }],              // multiplier on deduction magnitudes — bigger weight = harder hit
  misbehaviors:          [{ id, name, categoryId, basePoints, emoji }],
  records: [
    // Every record may also have optional v7+ fields: startTime ("HH:MM"), durationMin (number), note (string).
    // All three are user-editable post-hoc via the edit pencil on the recent-row; absent on legacy records.
    { id, personId, activityId,    timestamp, dateISO, points, kind: 'earn',   startTime?, durationMin?, note? },
    { id, personId, rewardId,      timestamp, dateISO, points, kind: 'redeem', startTime?, durationMin?, note? },
    { id, personId, misbehaviorId, timestamp, dateISO, points, kind: 'deduct', startTime?, durationMin?, note? }   // points is POSITIVE magnitude; balance subtracts
  ],
  comments: [  // in-app feedback backlog, one thread per page
    { id, page: 'log'|'dashboard'|'rewards'|'settings', text, done, createdAt, completedAt }
  ]
}
```

**Cost / points formulas (single source of truth):**
- Activity earn: `activityPoints(a) = Math.round(a.basePoints * activityCategory.weight)`
- Reward redemption: `rewardCost(r) = Math.round(r.cost * rewardCategory.weight)` — weight <1 encourages, >1 discourages.
- Misbehavior deduction: `misbehaviorPoints(m) = Math.round(m.basePoints * misbehaviorCategory.weight)` — returns POSITIVE magnitude; balance subtracts it. Bigger weight = harder hit (Disrespect ×3 stings hardest, mirrors Bonus ×3 on the earn side).
- All three are computed at log/redeem/deduct time and frozen on the record (`record.points`). Changing a category weight later does NOT mutate history.
- Balance = sum(earn points) − sum(redeem points) − sum(deduct points). **Can go negative** — by design, deductions always apply (no floor at 0).
- Streak = consecutive days with `(earn − deduct) > 0`. Pure-deduct days and break-even days reset the streak.

**Migration:** `migrate(data)` in `load()` chains v1 → v2 → v3 → v4 → v5 → v6 → v7.
- v1→v2: stamp `kind:'earn'` on records, seed `rewards` with defaults.
- v2→v3: add `rewardCategories` (6 defaults). Existing rewards keep their `cost` field; without a `categoryId` they're treated as weight 1 (no behavior change). New rewards via the modal get assigned a `categoryId`.
- v3→v4: add empty `comments` array.
- v4→v5: sync backend moved to GitHub Gist — `delete settings.syncUrl`, add `settings.gistId` + `settings.gistToken` (both default `''`).
- v5→v6: introduce misbehaviors — seed `misbehaviorCategories` (6 defaults, stable `mb-*` ids) and `misbehaviors` (10 defaults). Existing records need no rewrite; new `kind: 'deduct'` records appear going forward.
- v6→v7: optional per-record metadata fields — `startTime` ("HH:MM"), `durationMin` (number), `note` (string). All three optional; the migrator just bumps `version`. Edited post-hoc via `openRecordDetailsModal()` triggered by the edit pencil on recent rows. Legacy records lack these fields entirely.
- **MUST be called on remote payloads** before adopting — both `pullFromCloud()` and the *Save & test* handler do this. Skipping migration on remote data caused a real bug on 2026-05-17: cloud data pushed by a v1 client had no `rewards` array, so after adoption `state.rewards.length` threw and the *Add reward* button silently failed.

**Forward-compat hazard for v6:** an OLD v5 client pulling v6 cloud data would see `kind: 'deduct'` records and `isEarn()` (old version: `r.kind !== 'redeem'`) would mistakenly count them as earns. After deploying v6, refresh ALL of Jerry's devices on next use — don't leave a v5 tab open mid-transition. The v6 `isEarn` correctly excludes both `'redeem'` and `'deduct'`.

## Default seed (see `defaultData()` near top of `<script>`)

**Activity categories** with weights: Hygiene ×1, Learning ×2, Chores ×1.5, Kindness ×2, Self-care ×1, Bonus ×3. 25 starter activities (incl. "Cooperate with family" in Kindness and "勇於表達自己" in Bonus — Jerry-family-specific character items kept in defaults).

**Reward categories** with weights — deliberately nudge behavior: Bonding & rituals ×0.8, Social & friendships ×0.9, Skills & creativity ×0.9, Experiences ×1.0, Privileges ×1.0, Materials ×1.5. **21 default rewards** (EQ/experience-focused; explicitly NO material defaults — Materials category exists so the user can add their own and have them be naturally more expensive). Base costs span 15–500 pts; weighted actuals span 12 pts (bedtime story) to 500 pts (zoo day). Includes "Make ice cream with daddy" (80 base, Bonding ×0.8 = 64 actual) — a Jerry-family-specific reward kept in defaults.

**Misbehavior categories** with weights — mirrors the activity-side value system (the same nudge principle, applied to deductions): Disrespect ×3, Unkindness ×2, Avoiding learning ×2, Avoiding chores ×1.5, Hygiene neglect ×1, Self-care neglect ×1. **10 default misbehaviors** spanning all six categories (e.g. "Lied to parent" base 5 in Disrespect ×3 = −15; "Refused to brush teeth" base 3 in Hygiene neglect ×1 = −3). Category IDs are stable `mb-*` strings so migration adoption stays clean. Behavior intent: heavier deductions for the moral violations Jerry most wants to discourage (mirror image of Kindness/Bonus on the earn side).

Two placeholder kids ("Child 1", "Child 2"). Settings → Rewards has a **"Load 21 suggested rewards"** button that swaps the live rewards list with `defaultRewards()` so existing users (whose cloud may still have the older 10-reward seed, or v2 uncategorized rewards) can adopt the new defaults in one click.

## Code layout inside `<script>`

Sections are marked with `/* ---------- Name ---------- */` banners. Skim these instead of reading line-by-line:

| Section | What lives there |
|---|---|
| Storage | `load()`, `save()`, `saveLocal()`, `STORAGE_KEY`, `migrate()` |
| Utilities | `todayKey`, `startOfWeek`, `startOfMonth`, `periodCutoff`, `activityPoints`, `rewardCost`, `misbehaviorPoints`, `misbehaviorCategoryOf`, `personById`, `categoryOf`, `isEarn`, `isRedeem`, `isDeduct` |
| Theme | `applyTheme()` — `light` / `dark` / `auto` |
| UI: Person row & Hero | `renderPersonRow`, `renderHero`, `activePerson` |
| Activity grid | `renderActivityList` — renders by category |
| Misbehavior grid | `renderMisbehaviorList` — collapsed Behavior section at bottom of Log tab; red `.deduct-btn` styling; `#behavior-toggle` chevron expands the list |
| Logging | `logActivity`, `logMisbehavior` (confirm dialog + undo toast — no confetti), `renderRecent` — recent list shows earns / redeems / deducts; deducts display with red `−N` |
| Totals & filters | `earnedForPerson`, `redeemedForPerson`, `deductedForPerson`, `balanceForPerson` (earn − redeem − deduct, can be negative), `totalForPerson` (back-compat — periods other than 'all' return earn-only) |
| Rewards tab | `renderRewardsTab`, `renderRewardsPersonRow`, `renderRewardsHero`, `renderRewardsGrid`, `redeemReward`, `renderRedemptionsList` |
| Dashboard | `renderDashboard`, `renderStatsGrid` (shows net = earn − deduct for periods, full balance for all-time), `computeStreak`, `renderTopActivities`, `renderTopMisbehaviors`, `filteredEarnRecords`, `filteredRecords` |
| Charts | `renderCharts` + 5 individual chart renderers (Chart.js); trend charts filter to `isEarn` records |
| Settings | `renderPeopleList`, `renderCategoriesList`, `renderActivitiesList`, `renderRewardCategoriesList`, `renderRewardsSettingsList`, `renderMisbehaviorCategoriesList`, `renderMisbehaviorsSettingsList`, `renderSyncCard`, `applySettingsSection` (filters cards by `settingsSection` global) |
| Cloud sync (GitHub Gist) | `pushToCloud`, `pullFromCloud`, `queuePush`, `setSyncStatus`, `syncPayload`, `gistHeaders`, `extractGistId`, `GIST_API`, `GIST_FILENAME` |
| Comments / feedback | `renderCommentBar`, `openCommentDrawer`, `renderCommentList`, `postComment`, `toggleComment`, `deleteComment`, `commentsAsMarkdown`, `copyAllComments`; `currentPage` + `commentFilter` globals; `adjustDockPadding` |
| Modals | `openPersonModal`, `openCategoryModal`, `openActivityModal`, `openRewardCategoryModal`, `openRewardModal`, `openMisbehaviorCategoryModal`, `openMisbehaviorModal` |
| Export / Import | `btn-export` / `btn-import` handlers |
| Tabs / theme toggle / toast / confetti / helpers | self-explanatory at end |

## Conventions

- All state mutations call `save()` then a relevant `render*()`. Never write to `localStorage` outside `save()`.
- IDs are generated by `uid()` (timestamp + random). Never reuse or hardcode IDs.
- HTML is built by template literals; **always wrap user-provided text in `escapeHtml()`** to prevent XSS.
- Colors come from `PERSON_COLORS` / `CATEGORY_COLORS` palettes. Don't introduce ad-hoc hex values for new UI; pick from the existing CSS variables.
- Charts must call `destroyChart(name)` before re-rendering or memory leaks compound.
- Mobile-first: minimum tap target is ~38px. The bottom nav owes the bottom safe-area inset.

## Common edits — where to make the change

| Task | Edit point |
|---|---|
| Add a default activity | `defaultData()` → `activities` array |
| Add a default reward | `defaultRewards()` (called from `defaultData()`) |
| Add a default reward category | `defaultRewardCategories()` |
| Add a default misbehavior | `defaultMisbehaviors()` |
| Add a default misbehavior category | `defaultMisbehaviorCategories()` (stable `mb-*` ids) |
| Rename a category | Settings tab in the app (do NOT edit `defaultData` if a user already has data) |
| Add a new chart | Add `<canvas>` in dashboard tab + a `renderChart<Name>()` function called from `renderCharts()` |
| Change point math | `activityPoints()` for earnings; `rewardCost()` for redemptions; `misbehaviorPoints()` for deductions — single sources of truth |
| Change balance math | `balanceForPerson()` / `earnedForPerson()` / `redeemedForPerson()` / `deductedForPerson()` — only place to edit |
| Change storage location | `STORAGE_KEY` constant |
| Restyle | CSS variables in `:root` and `[data-theme="dark"]` at top of `<style>` |
| Bump schema | Increase `version` in `defaultData()`, add a migration block in `migrate()` |

## Tab layout

Tabs in the bottom nav: **Log** (earn + Behavior section for deductions), **Dashboard** (charts + stats), **Rewards** (redeem), **Search**, **Settings** (CRUD + sync). Tab sections are `<section class="tab" id="tab-{name}">` and shown/hidden by the bottom-nav click handler. When adding a new tab, also update `renderAll()` so all visible tabs re-render after a state change.

**Settings sub-sections (v7+):** the Settings tab has a horizontal pill strip at the top filtering visible cards into 5 buckets: People, Earn (categories + activities), Rewards (reward categories + rewards), Behavior (misbehavior categories + items), Data (export/import + cloud sync + danger zone). Each `.settings-card` carries a `data-settings-section="..."` attribute; `applySettingsSection()` toggles visibility based on the `settingsSection` global. When adding a new settings card, tag it with the right section.

**Log tab structure:** person row → hero card → activity grid (by category) → recent list → collapsed "Behavior (deductions)" section. The Behavior toggle (`#behavior-toggle`) shows/hides `#misbehavior-list`; default state is collapsed so positive actions stay front-and-center.

The bottom nav and the comment bar live together inside a single fixed `.bottom-dock`. The dock's height is measured at runtime by `adjustDockPadding()` (called on boot + resize) which sets `body { padding-bottom }` so content never hides behind it — don't hardcode that padding.

## Comments / feedback backlog (v4)

An Instagram-style comment bar (the pill in `.comment-bar`) sits just above the four tabs on every page. The badge shows `openCommentCount(currentPage)` — count of not-done comments for the active page. `currentPage` is set in the bottom-nav click handler; if you add navigation elsewhere, update it and call `renderCommentBar()`.

Not-done comments are editable: each open comment has a pencil button that swaps the text for an inline textarea + Save/Cancel (Cmd/Ctrl+Enter saves, Esc cancels). Done comments are locked — uncheck first to edit. `editingCommentId` is a module-scope global; only one comment can be in edit mode at a time.

Tapping the pill opens `#comment-drawer-backdrop` (a slide-up sheet, z-index 105). Each page keeps its own thread (`comments[].page`). The drawer = scrollable `#comment-list` (flex:1) + pinned `.comment-input-bar` (auto-growing `<textarea>`, capped 120px). Posting appends to the bottom and scrolls down; Cmd/Ctrl+Enter posts, plain Enter is a newline (mobile-friendly). Each comment row has a checkbox (`toggleComment` → strikethrough + `completedAt`) and a delete button. Open/All filter is `commentFilter`. The ⧉ button calls `copyAllComments()` → `commentsAsMarkdown()`, a checklist grouped by page (`## Log` / `## Dashboard` / `## Rewards` / `## Settings`, `- [ ]`/`- [x]`, newlines flattened), copied via `copyText()` (clipboard API with execCommand fallback). Comments persist in `state.comments` and sync like everything else.

## What NOT to do

- **Don't add npm/build tooling.** The file is intentionally one-file, no-build. Just open it in a browser.
- **Don't add a paid backend** or any service with a monthly cost. Auto-sync uses a free, user-owned GitHub Gist (a classic PAT with the `gist` scope only). Runtime cost stays $0.
- **Don't read the whole 1880-line file** to make a small edit. Grep for the section banner, edit, save.
- **Don't migrate the localStorage schema** without bumping `version` and writing a migrator in `load()`.
- **Don't mutate `record.points` retroactively** when category weights change. Past entries are immutable history.

## Offline / fully self-contained option

The file loads Chart.js from `cdn.jsdelivr.net`. If Jerry needs true offline use (e.g. a flaky-WiFi tablet for the kids), download `chart.umd.min.js` once and replace the `<script src="...">` tag with `<script>…inlined content…</script>`. Adds ~70 KB to the file (~145 KB total — still tiny).

## Token-efficiency rules for future Claude sessions

1. **Read this file first**, then jump straight to the relevant section banner in `family-rewards.html`. Don't dump the whole file into context.
2. **Use Grep** to locate functions before reading. e.g. `Grep "function renderDaily"` instead of scanning.
3. **Make targeted Edits**, not rewrites. The Edit tool's diff costs less than a full Write.
4. **Don't re-derive** the schema, defaults, or sync model — they're documented above. Trust this file; verify only when the user reports something that contradicts it.
5. **When the user asks for a small change**, propose a 1–3 line diff, don't restate the architecture.

## Verification done at build time

**Test suites kept in the session scratchpad during dev** (not committed, no test framework dep). Re-run pattern: read the HTML, regex-extract the inline `<script>`, strip from `state = load();` onward, `vm.runInContext` with localStorage + DOM/`fetch`/`setTimeout` stubs. The element stub needs `querySelector`/`appendChild` because the comment + reward render functions build DOM nodes.
- Core logic / rewards v3 / comments v4 — default shape, weighted points math, per-period totals, streak, balance, save/load roundtrip, chained migrations, comment post/toggle/delete + markdown export.
- `verify_gist.js` (GitHub Gist sync, 2026-05-26) — **20 logic checks, all passing**: `extractGistId` (URL + raw id), v5 default shape, v4→v5 + chained v1→v5 migration, `syncPayload` strips `gistToken`/`gistId` from the uploaded copy, first-device POST creates a gist and captures its id, existing-id PATCH, pull adopts cloud + preserves this device's credentials, and stale-cloud/newer-local pushes instead of overwriting.

**Note:** the retired `sync_test.js` / `setup_bug_test.js` targeted the old Google Sheets / Apps Script backend; `verify_gist.js` supersedes them. The "don't `save()` before reading the gist" regression is covered by its migration + first-device branches.
