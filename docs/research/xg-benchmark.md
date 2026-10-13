# Public xG benchmark: what to compare against and how to join it

Resolves [#3](https://github.com/obaume/xg_model/issues/3). Researched 2026-10-13.

## Answer

- **Benchmark: MoneyPuck shot-level data.** It's free for non-commercial use (credit required), covers every season from 2007-08 to the current 2026-27 (updated nightly), and has one row per Fenwick shot with `xGoal`. It's also well calibrated: season `xGoal` totals land within about 2.5% of actual goals in both seasons we checked.
- **Join key: `(game_id, period, game seconds, shooter NHL player id)`.** First build the full NHL game id as `season * 1_000_000 + game_id`. **No event id lines up.** MoneyPuck's `id` is the event number (`#`) from the NHL HTML play-by-play report. That isn't the NHL API's `eventId` or `sortOrder`, and it isn't sportsdataverse's `event_idx`/`event_id`. When we checked, 99.6–99.8% of MoneyPuck shots matched exactly one sportsdataverse shot on this key.
- **Second reference, free and already in our feed:** the sportsdataverse `nhl_pbp_full` release has an `xg` column (fastRhockey's published boosters). You don't need a join to use it, but it's badly calibrated (+12% to +42% over actual goals), so it's only a rough cross-check.
- **Evolving-Hockey, Natural Stat Trick, HockeyViz:** none of them offers an open per-shot xG download that we could confirm. Evolving-Hockey is paywalled and its ToS forbids redistribution. Natural Stat Trick has free team, player and game-level xG, and allows rate-limited scraping with a free key. HockeyViz is subscriber-supported and has no data download.

## 1. MoneyPuck shot-level data

### Seasons and format

Source: <https://moneypuck.com/data.htm>.

- Each season is a zip holding one CSV, at `https://peter-tanner.com/moneypuck/downloads/shots_{YEAR}.zip`. `{YEAR}` is the year the season **starts**: `shots_2025.zip` is 2025-26.
- The single-season files run from `shots_2007.zip` to `shots_2026.zip`. The page says 2026-27 is "updated nightly". On 2026-10-13, `shots_2026.zip` returned `Last-Modified: Tue, 13 Oct 2026 07:40:16 GMT` (1.2 MB).
- There are also combined files: `shots_2007-2025.zip` and `shots_2021-2025.zip`.
- One zip holds the regular season and playoffs (`isPlayoffGame`). Shootouts are excluded: in 2025-26 the only period-5 rows are 52 playoff double-overtime shots, and there are no regular-season period-5 rows.
- Data dictionary: <https://peter-tanner.com/moneypuck/downloads/MoneyPuck_Shot_Data_Dictionary.csv>.

### Columns

The dictionary was checked against the real `shots_2025.csv`: 124 columns, 119,271 rows, 1,394 games (game ids 20001 to 30416). The rows are SHOT 68,896, MISS 41,810 and GOAL 8,565, so it's **Fenwick only** (no blocked shots).

| Need | Column(s) | Notes (from dictionary + inspection) |
|---|---|---|
| Their xG | `xGoal` | Probability the shot is a goal. Siblings are `xRebound`, `xFroze`, `xPlayStopped`, `xPlayContinuedInZone`, `xPlayContinuedOutsideZone`, `xShotWasOnGoal`. |
| Game | `game_id`, `season` | `game_id` is the **short** NHL id, `{gameType:02}{number:04}` (e.g. `20001`, `30416`), not the 10-digit `2025020001`. `season` is the start year (`2025`). |
| Event | `id`, `shotID` | `id` is "the event # of the shot in the game". It equals the `#` column of the NHL HTML report (`PL020001.HTM`), as shown below. `shotID` is just a row counter. |
| Time | `period`, `time` | `time` is seconds since the start of the game, i.e. `(period-1)*1200 + seconds into period`. |
| Shooter / goalie | `shooterPlayerId`, `shooterName`, `goalieIdForShot` | NHL player ids, the same id space as the NHL API. |
| Coordinates | `xCord`, `yCord` (raw), `xCordAdjusted`/`yCordAdjusted` (attacking right), `arenaAdjusted*` | The raw coordinates are identical to the NHL API's `xCoord`/`yCoord` in 2025-26 (99.7%), but only about 88% identical in 2018-19. |
| Teams / state | `teamCode`, `homeTeamCode`, `awayTeamCode`, `isHomeTeam`, `homeSkatersOnIce`, `awaySkatersOnIce`, `homeEmptyNet`, `awayEmptyNet` | Use these for team totals and strength splits. |
| Outcome | `event` (SHOT/MISS/GOAL), `goal`, `shotWasOnGoal` | |

The dictionary also says: "The data has been collected from several sources including the NHL and ESPN" and "No guarantees are made to the quality of the data."

### Terms of use

These quotes are verbatim from <https://moneypuck.com/data.htm>:

- "The data below is free to use for non-commercial purposes and by journalists for ad-hoc use."
- "Please clearly credit MoneyPuck.com in all cases where you are showing anything using our data as an input."
- "For other purposes please inquire by messaging" (followed by a contact link)
- "Non-approved scraping of the MoneyPuck website will be blocked." and "Please message us for approval before scraping any data not listed on this page."

The dictionary adds: "You are welcome to use this data in any work. Just please cite MoneyPuck.com".

**What this means for us:**

- Downloading the listed zip files is the sanctioned path. Don't scrape any other MoneyPuck page.
- Our uses (learning, portfolio, personal) are non-commercial, so they're fine as long as every chart or table that uses it credits MoneyPuck.com.
- **Don't commit or bundle MoneyPuck data in the package.** The map already says consumers get the package, not data files. The comparison should fetch the files when it runs.
- If the project ever turns commercial, ask MoneyPuck first.

### Other MoneyPuck files (for team/player totals)

These are also on the data page. Their xG columns are documented in the player data dictionary (<https://peter-tanner.com/moneypuck/downloads/MoneyPuckDataDictionaryForPlayers.csv>), which I didn't inspect.

- Season summaries: `moneypuck/playerData/seasonSummary/{YEAR}/regular/{skaters,goalies,lines,teams}.csv` (2008-09 to 2026-27).
- Game-by-game: `downloads/seasonPlayersSummary/{skaters,goalies,lines}/{YEAR}.zip` and `moneypuck/playerData/careers/gameByGame/all_teams.csv`.

You don't need these for the sanity check: summing `xGoal` from the shot file by `teamCode` or `shooterPlayerId` gives the same aggregates, and keeps the shot set identical to ours.

## 2. Join keys to sportsdataverse / NHL API play-by-play

### Identifiers on each side

| Source | Game id | Event identifiers |
|---|---|---|
| NHL API `api-web.nhle.com/v1/gamecenter/{id}/play-by-play` | `id` = `2025020001` | `eventId` (not sequential, e.g. 71, 102, 80) and `sortOrder` (sequential with gaps, e.g. 29, 35, 38) |
| NHL HTML report `nhl.com/scores/htmlreports/20252026/PL020001.HTM` | in file name | `#` (e.g. 14, 19, 22) |
| sportsdataverse `nhl_pbp_full` (`load_nhl_pbp_full`) | `game_id` = `2025020001`, `season` = `"20252026"` | `event_idx` = `event_id` (e.g. 21, 27, 30), which is sportsdataverse's own sequence |
| MoneyPuck | `game_id` = `20001`, `season` = `2025` | `id` (e.g. 14, 19, 22), which is the HTML report `#` |

This comes from the first three Fenwick shots of game 2025020001 (CHI @ FLA, 2025-10-07):

| HTML `#` | MoneyPuck `id` | NHL API `eventId` | NHL API `sortOrder` | sportsdataverse `event_id` | Shooter | Period time |
|---|---|---|---|---|---|---|
| 14 | 14 | 71 | 29 | 21 | Rodrigues (8478542) | 1:16 |
| 19 | 19 | 102 | 35 | 27 | Forsling (8478055) | 1:35 |
| 22 | 22 | 80 | 38 | 30 | Bedard (8484144) | 1:44 |

MoneyPuck's `id` matched the HTML `#` for all 15 rows I compared by hand. It didn't equal any of the other three identifiers. The NHL API JSON doesn't carry the HTML `#` at all.

sportsdataverse's `event_idx` skips numbers (2, 3, 4, 5, 8, ...) because it counts shift (`CHANGE`) rows in its merged event+shift stream. Source: `load_nhl_pbp_full` schema in [`sportsdataverse/nhl/nhl_loaders.py`](https://github.com/sportsdataverse/sportsdataverse-py/blob/main/sportsdataverse/nhl/nhl_loaders.py). The values above were checked in `play_by_play_2026.parquet`.

### Recommended join

```text
mp.full_game_id = mp.season * 1_000_000 + mp.game_id     # 2025 * 1e6 + 20001 = 2025020001
ON  mp.full_game_id    = sdv.game_id
AND mp.period          = sdv.period
AND mp.time            = sdv.game_seconds                # MoneyPuck seconds-from-game-start
AND mp.shooterPlayerId = sdv.event_player_1_id
WHERE sdv.event_type IN ('SHOT','MISSED_SHOT','GOAL') AND sdv.period_type != 'SHOOTOUT'
```

**Tie-break.** A few keys have more than one candidate, typically when the same shooter takes two attempts in the same second. Rank the candidates inside each key group, MoneyPuck by `id` and sportsdataverse by `event_idx` (or NHL `sortOrder`), and pair them by rank. Coordinates are a secondary check.

**Measured results** (scripts run against the real files, not committed):

| Season | MoneyPuck shots | sportsdataverse Fenwick (no shootout) | Unmatched MoneyPuck | MoneyPuck shots with >1 candidate |
|---|---|---|---|---|
| 2018-19 | 117,622 | 117,727 | 270 (0.23%) | 395 (0.34%) |
| 2025-26 | 119,271 | 119,357 | 438 (0.37%) | 461 (0.39%) |

- For game 2025020001 against the raw NHL API, all 82 MoneyPuck shots matched exactly one API shot, with identical coordinates.
- The unmatched rows cluster in individual games, such as several late third-period shots in 2025020660. That looks like the two feeds disagreeing within a game, not a flaw in the key. Report the unmatched count instead of hiding it.

### Season-label gotcha

Three conventions are in play for the same season:

- MoneyPuck `season = 2025` and file `shots_2025.zip` (start year).
- The sportsdataverse release asset is `play_by_play_2026.parquet` (end year), and its `season` column is `"20252026"`.
- The NHL game id prefix is `2025` (start year).

## 3. Other per-shot or per-game xG sources

| Source | Granularity | Cost | Terms | Verdict |
|---|---|---|---|---|
| **sportsdataverse `nhl_pbp_full` `xg` column** | Per shot, already in our feed | Free; repo is MIT ([sportsdataverse-data](https://github.com/sportsdataverse/sportsdataverse-data)) | Model code is a port of fastRhockey (Apache-2.0), per the [`nhl_xg.py`](https://github.com/sportsdataverse/sportsdataverse-py/blob/main/sportsdataverse/nhl/nhl_xg.py) docstring. Two boosters (5v5 / special teams) plus a penalty-shot constant. | Secondary cross-check only. It's miscalibrated: on shots matched to MoneyPuck, total `xg` was 11,214 vs 7,917 goals in 2018-19 (+42%) and 9,374 vs 8,337 in 2025-26 (+12%). Fenwick `xg` is null for about 2% of shots in 2025-26. |
| **Natural Stat Trick** | Team, player, line and game level xG on the website. I found no per-shot export. | Free site; Patreon for extra features | [Scraping policy](https://www.naturalstattrick.com/scraping.php): "Automated bot and scraping traffic is permitted on data.naturalstattrick.com and requires an access key". Keys are free with an account, need approval, and are rate-limited to "80 page loads in 5 minutes, and 180 page loads in 1 hour". Other subdomains are "intended for purely human use". | Optional third opinion on team/player season totals. Not a per-shot source. |
| **Evolving-Hockey** | Per-game and per-player xG on the site | Subscriber content via Patreon | [ToS](https://evolving-hockey.com/tos/) §7: "You agree not to reproduce, re-distribute, copy, sell, trade or otherwise use any material obtained from the Website", with use "prohibited except as covered by fair use". | Not usable as an automated benchmark. At most, eyeball it manually. |
| **HockeyViz** (Micah McCurdy) | Team and player xG rates in charts | Subscriber-supported | No data licence or download found on <https://hockeyviz.com/>. | Not usable. |

## 4. How to use it in validation (input for #7)

- **Per shot.** Join our Fenwick shots to MoneyPuck with the key above, then report the Pearson correlation of our `xG` against `xGoal`, along with the match rate.
  - For scale: MoneyPuck vs the sportsdataverse `xg` model correlates at only **0.65 (2025-26) and 0.75 (2018-19)** per shot. Two reasonable public models differ that much shot by shot, so a per-shot correlation around 0.7–0.9 isn't alarming.
  - Compare calibration on the matched set: sum of `xG` against goals, ours and MoneyPuck's. MoneyPuck: 8,488 vs 8,337 goals (+1.8%) in 2025-26, and 7,718 vs 7,917 (−2.5%) in 2018-19, on shots that also have sportsdataverse `xg`.
- **Season totals.** Group both sides by `teamCode` and by shooter id.
  - For scale: MoneyPuck vs sportsdataverse team-season xG correlates at 0.96–0.99, and player-season xG (50+ shots) at about 0.99. Because these aggregates are driven by shot volume, a correlation under about 0.95 would mean something is wrong.
- **Strength states.** Split by MoneyPuck `homeSkatersOnIce`/`awaySkatersOnIce`/empty-net to compare each of our four strength-state models separately.
- Don't use MoneyPuck as the calibration gate. Calibration against actual goals is the gate (see #1). MoneyPuck is a sanity check, and its features (arena adjustment, ESPN-sourced data in some seasons) differ from ours.

## Sources

- MoneyPuck data page and terms: <https://moneypuck.com/data.htm>
- MoneyPuck shot data dictionary: <https://peter-tanner.com/moneypuck/downloads/MoneyPuck_Shot_Data_Dictionary.csv>
- MoneyPuck shot files inspected: `shots_2025.zip`, `shots_2018.zip` (downloaded 2026-10-13)
- NHL API play-by-play: <https://api-web.nhle.com/v1/gamecenter/2025020001/play-by-play>
- NHL HTML play-by-play report: <https://www.nhl.com/scores/htmlreports/20252026/PL020001.HTM>
- sportsdataverse loaders and schema: <https://github.com/sportsdataverse/sportsdataverse-py/blob/main/sportsdataverse/nhl/nhl_loaders.py>
- sportsdataverse xG module: <https://github.com/sportsdataverse/sportsdataverse-py/blob/main/sportsdataverse/nhl/nhl_xg.py>
- sportsdataverse release files inspected: <https://github.com/sportsdataverse/sportsdataverse-data/releases/tag/nhl_pbp_full> (`play_by_play_2026.parquet`, `play_by_play_2019.parquet`)
- Natural Stat Trick scraping policy: <https://www.naturalstattrick.com/scraping.php>
- Evolving-Hockey terms: <https://evolving-hockey.com/tos/>
- HockeyViz: <https://hockeyviz.com/>
