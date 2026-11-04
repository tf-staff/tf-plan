# TF-Plan — Trello Power-Up

Countdown-and-move Power-Up for the UC Davis Tech Foundry (Davis/GBSF + Aggie Square, Sacramento). Engineers put a timer on a Trello project card; when it elapses, the card (or a link-card) is moved automatically — modeling manufacturing hand-offs (e.g., card sits in a machine queue for 2h, then returns to the engineer's column or goes to post-processing). Also carries team workload reporting. Internal tool, used daily by TF staff.

**Current version: 4.2** — the whole Power-Up is the single file `tf-plan.html`, served by GitHub Pages from this repo (`https://tf-staff.github.io/tf-plan/tf-plan.html`). No build step: edit the file, bump `APP_VERSION`, push.

## API key

The real Trello API key was restored in v4.2 (it had been a placeholder since the v4.0 upload). `TOKEN_KEY` is derived from `APP_KEY`, so changing the key orphans stored tokens — every machine must re-authorize. The key lives in the Trello Power-Up admin at trello.com/power-ups/admin (API key tab); it's public-safe by design.

## Release workflow (agreed with owner — standing permission)

The owner (Steven) is not a developer — explain everything in plain language.

1. Make the change, then **bump `APP_VERSION`** (every push, no exceptions).
2. Extract the script and run `node --check` on it.
3. Load the real `tf-plan.html` in headless Chromium via Playwright and confirm it boots without errors.
4. Commit with the version first: `v4.1: add X`. **Push straight to `main`** — that's what goes live on GitHub Pages. No PR, no owner review.
5. Report back plainly: what changed, and that it's live.

**Rollbacks** — git history is the backup system. **Never rewrite or delete history** (no force-push, no reset of `main`).
- "Go back to 4.0" → restore that version's `tf-plan.html` (`git checkout <commit> -- tf-plan.html`), bump to a new version number, push as a new commit.
- One bad change in a release → `git revert` just that change; keep the rest.
- When reporting any revert, remind the owner: card data on Trello is unaffected, but an older version may not display fields added after it.

## Non-negotiable conventions

1. **Bump `var APP_VERSION` on every functional or visual change** (e.g. v4.0 → v4.1). It renders after the name in the card-back section title and dashboard popup title (Steven's deployment cache-check), and it cache-busts internal iframe URLs (`?mode=panel&v=`). Trello caches connector iframes aggressively; mixed-version connector/panel caused a "perpetually moving…" bug once. Suggest bumping the registered connector URL's `?v=` in the Power-Up admin on releases too.
2. **Visual language matches the TF Power-Up suite** (TF-Recharge is the reference): dark `#242528`/`#2b2c2f` surfaces, `#579dff` blue accent with dark button text, `--line #46474b` hairlines, uppercase letter-spaced micro-labels (`.lbl`), tabular numerals, `.savechip` status pills, `.seg` segmented controls, `.quick` preset rows. Theme-adaptive via `data-color-mode` on `<html>` with late-attribute retries (Trello sets it late) and a light palette.
3. **No red text anywhere, and no blue highlight on the active-timer box** (Steven removed both in v3.4). Warning/error states use yellow (`--yellow`); the active box is neutral (`--subtle` bg + `--line` border). Countdown/progress/Start keep the blue accent.
4. **All icons are the small clock** — one SVG generated in code (`clockSvgMarkup`): inline `currentColor` copies in UI, data-URI copies for Trello icon slots (board button light/dark pair, badge, card-back section).
5. **Every card move lands at the top of the destination column** (`&pos=top`).
6. **Simulation-first development**: logic changes are verified in a Node harness before shipping (stub `t.get/t.set`, stub `fetch`, run the scan loop; see Version history for what the harnesses covered). Extract the script, `node --check` it, then run lifecycle simulations.

## Architecture — single-file, four-mode chassis

`?mode=` query param routes at boot:
- **connector** (default, no param): `TrelloPowerUp.initialize` — capabilities `board-buttons`, `card-badges`, `card-back-section` (`card-buttons` deliberately unused; disable it in the admin). Hosts the **scan loop**.
- **panel**: the persistent card-back section iframe — the primary interface. Mode seg (Move | Boomerang | 2-Step | Link Card), destination select(s), duration presets `30m/1h/2h/4h` + inline h/m custom cells (minutes may exceed 60; total = h×60+m) + inline blue Start; when armed: live 1s countdown (h:mm:ss), depleting progress bar with total duration pinned right, ETA, set-by, Cancel. Faint rounded background (`.panelwrap`); `overflow:hidden` + `t.sizeTo('#app')` (sizeTo excludes margins — margins on the wrapper caused a scrollbar once; keep margins 0).
- **dashboard**: board-button popup (height 560). Top to bottom: Mover heartbeat row, this-browser auth row (+ authorize/de-authorize, "as <name>" via `/members/me`), **My workload** (1–10 slider with Light/Moderate/Busy/Heavy/Overloaded label, 280-char note, "I need a check-in" checkbox, Update), **Team workload** (sorted check-ins first then heaviest; colored bars green ≤4 / blue 5–6 / yellow 7+; stale >4h ages shown yellow), Active timers table (with cancel), Recent moves log.
- **authreturn**: top-level window Trello redirects to after Allow; extracts token from URL fragment and hands it back (see Authorization).

Boot is guarded (framed + `typeof window.TrelloPowerUp !== 'undefined'` with a 3s retry for slow CDN); bare-URL opens show a styled setup page instead of throwing. `authreturn` dispatches before the guard — it legitimately runs top-level.

## Execution model (client-side only — no backend)

Power-Ups only run while a board tab is open. TF always has ≥1 machine with the board open (kiosk), so the **connector runs a 15s scan loop** (`ensureScanLoop` keeps the freshest `t` from any capability callback). Each pass: writes `tfplan_health` heartbeat, runs the workload nudge check, then executes due stages if this client holds a token.

**Claim protocol** (multi-machine race protection): write `claimedBy/claimedAt` → settle 900ms → re-read → first writer wins; claims older than 120s are stolen (dead client); failed moves release the claim. The panel also executes immediate stages locally when the setting browser is authorized (instant Boomerang/2-Step/Link hop); otherwise the kiosk sweeps it up within ~15s.

**Self-diagnosis**: a stage >45s overdue shows a yellow line in the panel naming the culprit via the heartbeat (no mover / mover unauthorized / mover error / retrying). A 401 on any REST call clears the stored token immediately.

## Data model (board-level `t.get/t.set('board','shared',…)`)

- `tfplan_index`: `{ [cardId]: { v:3, mode:'move'|'boomerang'|'2step'|'link', cardName, setByName, createdTs, durationMin, sourceUrl?, linkCardId?, stages:[{ action?:'move'(default)|'create-link'|'delete-card', destListId, destListName, at, verb, sym? }], claimedBy?, claimedAt? } }` — stages execute front-to-back when `at <= now`; executed stage is shifted off; entry is deleted when empty. `normEntry()` upgrades legacy v2 single-destination entries.
  - move: `[dest @ now+d]` · boomerang: `[dest @ now], [origin @ now+d]` · 2step: `[A @ now], [B @ now+d]` · link: `[create-link @ now]` + (unless indefinite ∞) `[delete-card @ now+d]`
- `tfplan_log`: newest-first, capped 50: `{ cardId, cardName, destListName, setByName, act:'moved'|'linked'|'unlinked', movedTs }`
- `tfplan_health`: `{ clientId, ts, ver, authed, lastError }` — written every scan pass
- `tfplan_workload`: `{ [memberId]: { name, level:1-10, note, checkin:bool, updatedTs } }`

Link-card specifics: created via `POST /1/cards?idList=…&name=<bare short URL https://trello.com/c/{shortLink}>&cardRole=link&pos=top`; created id saved as `linkCardId` for the timed `DELETE`. Cancel on a live link entry doesn't orphan the link-card — it reschedules the delete-card stage to *now* (`buildCancel`). Indefinite (∞ preset, link mode only) = single create stage, entry retires immediately, nothing tracked after.

Workload nudge: opt-in by first report; only the viewer's own staleness nudges them; fires `t.alert` toast when their report is >4h old, repeated at most hourly (localStorage key `tfplan_wlnudge_<board>_<member>`); updating resets the clock. Works only while that person has the board open.

## Authorization (DIY, partition-proof — do not use `t.getRestApi()`)

`getRestApi()` broke under browser storage partitioning (popup and iframe see different localStorage buckets). TF-Plan owns the flow: `beginAuthorize` opens `https://trello.com/1/authorize?...&response_type=token&callback_method=fragment&return_url=<self>?mode=authreturn`; the return page extracts the token with `/[#&]token=([^&#?]+)/` (**not** hex-only — modern tokens start `ATTA` and are alphanumeric; a hex regex once stored a 1-char token causing instant green→red pill flapping), `postMessage`s it to the opener (same-origin, partition-proof), writes localStorage as fallback, closes itself. Token stored at `localStorage['tfplan_token_' + APP_KEY]` in the trello.com partition, shared by connector/panel/dashboard iframes. Scope is the string `'read,write'` (object form is rejected: "invalid scope"). Requires the GitHub Pages origin (`https://tf-staff.github.io`) in the Power-Up admin's **Allowed Origins**.

Timers can be *set* by anyone; only machines that will *execute* (kiosk, ideally each engineer's machine for instant hops) need to authorize via the dashboard. Plan of record: kiosk authorizes as a **shared TF service account** so automation survives password changes — not yet done as of handoff.

## Version history

1.0 popup editor, per-card countdown + REST move · v2 persistent card-back panel, Recharge design language, board-level index, scan loop + claim protocol, DIY auth, clock icons · v3.x staged sequences (Boomerang default, 2-Step), pos=top, progress bar, heartbeat + overdue diagnosis, Link Card (`cardRole=link`, short-URL name, cancel-reschedules-delete, ∞ indefinite), no-red/neutral restyle, versioned titles/URLs · v4.0 workload reporting + 4h nudge. Harness-verified: boomerang lifecycle, link create→delete lifecycle + cancel semantics, nudge thresholds (opt-in, 4h, hourly cap, never for others' entries), duration math (90m = 1h30m).

## Verified / open at handoff

Field-verified: Move, Boomerang, badges, countdown/progress, auth flow, heartbeat. **Open:** (1) field-verify auth + moves after the v4.2 key restore; (2) link-card click-through — v3.5's `cardRole=link` rendered properly but opened a new window; v3.6 switched the name to the bare short URL to match manual copy-link behavior — **unconfirmed**; if it still opens a new window, try `cardRole=mirror` next (mirrors navigate natively and live-display the original); (3) kiosk service-account authorization; (4) v4.0 workload UI untested in the field; (5) possible future: orphan link-card sweeper (needs a registry of created ids), board-scoped duration presets.
