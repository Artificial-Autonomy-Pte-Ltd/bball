# bball — Masters Team Strategy App

A single-page, no-build web app for planning game strategy and optimizing lineups &
rotations for a masters/veterans basketball team. Built for a squad of ~47-year-olds where
**managing minutes matters as much as talent** — the rotation optimizer weights playing time
by each player's fitness/stamina so the tired guys rest and the floor unit stays balanced.

No login, no database, no build step. Just static files you can host on GitHub Pages.

## Features

- **Get In (headline feature)** — a mobile-first onboarding flow so teammates add themselves in
  ~60 seconds: name + jersey number, position, tactile sliders for each skill, an availability
  toggle, and an **interactive half-court shot chart** where they tap to drop their favorite
  scoring spots (tap once = *like*, twice = *love*, a third time removes it). It finishes with a
  shareable summary card — profile, a mini shot chart of their hot zones, and a fun "archetype"
  (Sniper, Maestro, Engine…). This is the "ooh, cool" moment that drives the crew to give real input.
- **Roster** — add/edit/delete players with jersey number, position (PG/SG/SF/PF/C, or blank/TBD),
  an optional secondary position (`pos2`, e.g. a backup PG), ratings 1–10 for shooting, defense,
  playmaking, rebounding and basketball IQ, a fitness/stamina rating, an availability toggle,
  free-text notes, a school **house** (colored chip), and their saved hot zones. A separate
  **Next Gen** section lists team kids (juniors) — they're on the squad but excluded from all
  playing math (lineups, rotation, shooting board).
- **Houses — Morrison vs ROTW** — each player belongs to a school house (Morrison = Blue,
  Buckley = Green, Hullett = Black). The Houses tab splits the squad into **Morrison** vs
  **ROTW** (Rest of the World = Buckley + Hullett) for casual intra-squad games, with a live
  quarter-by-quarter scoreboard (per-quarter and overall leader, in house colors).
- **Lineup builder** — pick any 5 available players; see the unit's average ratings and
  automatic strength/gap flags (e.g. *"low rebounding"*, *"no rim protector"*, *"low-stamina
  unit — rotate out early"*).
- **Rotation / minutes optimizer** — enter game length and number of segments; get a segment-
  by-segment rotation grid, total minutes and % per player, and a per-segment unit list. Includes
  an optional **Press mode** (see *Press mode* below). See *How the optimizer decides* below.
- **Shooting Room** — players log solo drill results (Free Throws by default, plus 3PT, Mid-range,
  Layups, Floaters) as makes/attempts with a date; the app computes the % and shows a competitive
  **team leaderboard** (🥇🥈🥉, ranked by overall % for the drill, ties broken by volume). A great
  traction hook — the uncles will chase the top spot.
- **Strategy board** — game plan, opponent notes, plus a reusable **Set Plays & Tactics** library
  (seeded with a Full-Court Press tactic: press type, trigger, trappers, safety, notes) that you
  can toggle active per game.
- **Live shared data** — reads the roster from a Google Sheet published as CSV so team members
  update their own info and the owner always sees current data. Falls back to local sample data.
- **Export / Import JSON** — nothing is stored in the browser; save/share state (including hot
  zones and set plays) as a file.

## How the shot data is stored

Each player carries a `hotZones` array. Every marked spot is one object:
`{ x, y, w, zone }` where `x` and `y` are **normalized 0–1 coordinates** on the half-court
(resolution-independent, so the chart redraws correctly at any size), `w` is the weight
(`1` = like, `2` = love), and `zone` is a human-readable label the app derives from the
coordinates (e.g. `left-corner-3`, `top-of-key-3`, `free-throw`, `at-the-rim`).

- In **JSON** export/import the full array (coords + weight + zone) round-trips exactly.
- In the **Google Sheet** it collapses to a single `hot_zones` text column: a comma-separated
  list of zone labels, with `!` appended for a "love" spot — e.g.
  `left-corner-3!, top-of-key-3, right-elbow`. Reading that column back rebuilds zone labels and
  weights (exact x/y aren't stored in the sheet, only the named zones).

## Practice logs (Shooting Room)

Each player also carries a `practice` array of sessions: `{ date, drill, makes, attempts }`. The
Shooting Room's leaderboard aggregates these per drill. This data is **time-series-ish**, so it is
kept in the app and in the **JSON export/import only** — it is *not* forced into the roster Google
Sheet, which stays a clean one-row-per-player table. Back it up with **Export JSON**. (If you ever
want a single latest-FT% figure visible in the Sheet you can add an optional `ft_pct` column by
hand, but the app doesn't read or require it — don't overcomplicate it.)

## Running it

Just open `index.html` in a browser, or host the folder (see *Deploy to GitHub Pages*).
It starts in **local mode** with sample players so you can try everything immediately.

## Live-data flow (Google Sheet → published CSV → app)

The app never talks to Google's API directly — it just fetches a **published CSV** URL, which
needs no auth and works from a static site. The flow:

1. **Create the sheet.** Use the provided `bball_roster_template.csv` (or the Google Sheet the
   setup created). The header row must be exactly:

   ```
   number, name, position, shooting, defense, playmaking, rebounding, iq, fitness, available, notes, hot_zones
   ```

   - `number` = jersey number (first column)
   - `position` = PG/SG/SF/PF/C (leave blank for TBD)
   - ratings = 1–10
   - `available` = TRUE/FALSE (also accepts yes/no/1/0)
   - `hot_zones` = comma-separated zone labels, `!` = a "love" spot (e.g. `top-of-key-3!, right-elbow`)

2. **Share for editing.** In Google Sheets: **Share** → add your teammates (or "Anyone with the
   link — Editor"). This is how players update their own rows.

3. **Publish to web as CSV.** In the sheet: **File → Share → Publish to web** → choose the tab →
   format **Comma-separated values (.csv)** → **Publish**. Copy the URL (it ends in
   `output=csv`). Publishing is separate from sharing: sharing lets people *edit*; publishing
   produces the read-only CSV the app *reads*.

4. **Wire it into the app.** Open the app → **Data** tab → paste the CSV URL → **Load from
   Sheet**. The banner turns green ("Live"). Click **Load from Sheet** again anytime to pull the
   latest edits. (Google's published CSV can lag edits by a minute or two — that's Google's
   cache, not the app.)

To go back to sample data, use **Use local sample data** on the Data tab.

### Importing the CSV template into Google Sheets

If you're starting from `bball_roster_template.csv`:
`sheets.google.com` → **Blank sheet** → **File → Import → Upload** → pick the CSV →
**Replace current sheet** → **Import data**. Then rename the sheet **bball roster** and follow
steps 2–4 above.

## How the rotation optimizer decides (transparent heuristic)

No black box. Three steps:

1. **Minutes targets by stamina.** Each available player gets a target number of minutes
   proportional to `fitness^1.5`, scaled so the targets add up to `5 × game length` (five players
   on the floor for the whole game). The `^1.5` means a fitter player gets meaningfully more run;
   nobody is targeted for more than the full game.
2. **Segment-by-segment greedy fill.** The game is split into equal segments. For each segment
   the app plays the 5 players with the most *minutes still owed per remaining segment* — so
   low-stamina players, who have small targets, naturally sit more often. Ties break toward
   whoever rested the previous segment (fairness).
3. **Balance pass.** If the chosen 5 has no PF/C (or no PG/SG) but a suitable player is on the
   bench with minutes left, the app swaps the lowest-need starter for them, so every segment
   keeps a big and a guard on the floor when your roster has them.

The result is a grid (● = on the floor), each player's total minutes and % of the game, and the
unit for each segment. Sub at each segment break.

### Press mode

Full-court pressing burns stamina fast, which matters a lot for a masters team. Tick **Press mode**
in the Rotation tab and pick an intensity (light / standard / heavy) and when to press (all game,
first & last segment, first half, last half). Press mode changes the math three ways, all shown in
the result:

1. **Harder fitness skew.** The minutes-target exponent rises from `1.5` to `2.0 / 2.5 / 3.0`
   (light / standard / heavy), so the fittest players get a much bigger share and low-fitness
   players get noticeably fewer minutes.
2. **Shorter stints.** A consecutive-segment cap pulls pressers off sooner; low-fitness players are
   capped at a single press-heavy segment before they must rest.
3. **Front/back-loading.** During press segments the fittest players are pulled onto the floor to do
   the trapping, while low-stamina players are steered to the calmer non-press segments and rested
   during the press. Pressing segments are marked 🔥 / red in the grid.

It still fields exactly 5 every segment and totals `5 × game length` player-minutes.

## Data & privacy

State lives **in memory only** — no localStorage/sessionStorage, no cookies, no server. Closing
the tab clears everything except what's in your Google Sheet. Use **Export JSON** to back up
your roster and strategy notes, **Import JSON** to restore them.

## Deploy to GitHub Pages (later)

This folder is already a clean static site (`index.html` at root). When you're ready:

1. `git init && git add . && git commit -m "bball app"` in this folder.
2. Create a GitHub repo (e.g. `bball`) and push:
   `git remote add origin <repo-url> && git push -u origin main`.
3. On GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch** →
   branch `main`, folder `/ (root)` → **Save**.
4. Your app appears at `https://<username>.github.io/bball/` within a minute or two.

The included `.gitignore` excludes Dropbox and OS junk (`.dropbox*`, `desktop.ini`, `.DS_Store`,
etc.) and the app's exported `bball-backup-*.json` files, so rosters aren't accidentally committed.

## Current squad

The app's default roster and the `bball_roster_template.csv` are seeded with the real 14-player
squad (jersey #1, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 19). Unknown skills are set to a neutral 5
so each player can tune his own in the **Get In** flow; only attributes a note clearly implies were
bumped. Everyone is available except **#11 Edwin** (not playing). A live Google Sheet titled
**bball roster** has been created with the same 13 rows and the `number`-first / `hot_zones`-last
columns — publish it to CSV and paste the URL into the Data tab to go live.

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app (HTML + CSS + JS, single file) |
| `bball_roster_template.csv` | The 13-player squad, ready to (re)import into Google Sheets |
| `README.md` | This file |
| `.gitignore` | Excludes Dropbox/OS junk and local backups |
