# What NHL play-by-play does sportsdataverse provide?

Research for [#2](https://github.com/obaume/xg_model/issues/2) (map: [#1](https://github.com/obaume/xg_model/issues/1)). Investigated 2026-10-13.

**Method.** Read the primary sources (the sportsdataverse-py loader source, the `nhl_pbp_full` release page and its assets, the fastRhockey R parser that derives the columns, the fastRhockey-nhl-data build code, and the raw NHL `api-web.nhle.com` feed). Then I checked every claim empirically. I installed `sportsdataverse==0.1.4` with `uv` on CPython 3.14.8 (Windows), downloaded all 18 `play_by_play_{season}.parquet` assets (2010–2027) and measured them with polars. Every figure without a link is my own measurement on those files, downloaded 2026-10-13. Source permalinks are pinned to the commits that were HEAD that day:

- sportsdataverse-py `1ef5af2`
- fastRhockey `24653b5`
- fastRhockey-nhl-data `ba888f6`

## TL;DR

The data is fit for Evolving-Wild-style Fenwick features. It covers 2009-10 to today with coordinates, a situation-code-derived strength state, empty-net and extra-attacker flags, on-ice player IDs, shot types and a daily refresh. Five things need handling, though:

1. **Season files are mislabelled.** 2014-15 is missing, files `2015`–`2018` hold the *following* season, and 2018-19 appears twice. Key on the `season` column or the `game_id` prefix, never the file name.
2. **`x_fixed` is not a real rink-side normalisation.** It is `abs(x)` with the sign forced by event team (home +, away −). `y_fixed` is just `y` for normal shots. The release drops `homeTeamDefendingSide`. Distance and |angle| are right for shots in the attacking half, but wrong for genuine long shots from the shooter's own half (mostly empty-net shots).
3. **Blocked shots are re-attributed to the non-shooting team.** `event_team_*` is the defending team, and `strength_state`, `x_fixed` and `shot_distance` follow that team's frame. The raw NHL feed attributes them to the shooter.
4. **CHANGE (shift) rows have `game_id = null`** in every season with shifts. On-ice and strength values on the *event* rows are unaffected. Drop CHANGE rows, or use `nhl_pbp_lite`, before any per-game lag.
5. **Recent games come from a separate, raw path.** The release refreshes once a day, around 08:00 UTC, with the previous day's games. The package's live per-game function (`nhl_web_pbp`) returns the *raw* NHL schema, not the release schema. To score a game before the next release, the project needs its own raw-to-release parser.

Install is fine: it installs cleanly with `uv` on Python 3.14, and the project is actively maintained.

## Seasons and coverage

The loader accepts seasons ≥ 2010, keyed by the **ending** year, so `2026` means 2025-26 ([`nhl_loaders.py` L44–178](https://github.com/sportsdataverse/sportsdataverse-py/blob/1ef5af272586e5d2c5873318e1f28d6a0d7bdbb3/sportsdataverse/nhl/nhl_loaders.py#L44-L178); [release notes](https://github.com/sportsdataverse/sportsdataverse-data/releases/tag/nhl_pbp_full): "season = the ENDING year"). The release holds 18 parquet files, 2010–2027. Each file contains regular season (`R`) and playoffs (`P`) only, no preseason.

**Mislabelled files.** The `season` column and `game_id` prefix of each file:

| File | Actually contains |
|---|---|
| `play_by_play_2010` … `_2014` | 2009-10 … 2013-14 (correct) |
| `play_by_play_2015` | **2015-16** |
| `play_by_play_2016` | **2016-17** |
| `play_by_play_2017` | **2017-18** |
| `play_by_play_2018` | **2018-19** |
| `play_by_play_2019` | 2018-19 (correct, so 2018-19 appears twice: 1,359 vs 1,358 games) |
| `play_by_play_2020` … `_2027` | correct |
| *(none)* | **2014-15 is absent** |

This is a known, open upstream defect: [fastRhockey-nhl-data#16](https://github.com/sportsdataverse/fastRhockey-nhl-data/issues/16). As a result, `load_nhl_pbp(2015)` silently returns 2015-16.

**Coordinates.** `x`/`y` are present on essentially every Fenwick shot in every season. Null share is ≤ 0.03% everywhere and 0.0% from 2011-12 on. (0,0) placeholders are ≤ 0.01%. Coordinates are reliable from 2009-10.

**Strength, on-ice players and shifts** (unblocked shots, excluding shootouts):

| Season (end year) | `strength_state` null | on-ice IDs null | notes |
|---|---|---|---|
| 2010 (2009-10) | 100% (column absent) | 100% (columns absent) | no CHANGE rows. `home_skaters`/`away_skaters`/`empty_net` are present |
| 2011–2019 | 0–0.08% | 0–0.08% | |
| 2020 (2019-20) | **14.2%** | **14.3%** | 174 regular-season games have no strength state at all |
| 2021–2027 | 0% | 0–0.6% | |

**Season sizes.** 2025-26 has 1,394 games (1,312 R + 82 P). A naive `n_unique("game_id")` reports 1,395 because the null `game_id` of CHANGE rows counts as a value (see below). It has 1.10M rows, of which 656k are CHANGE. It has 119k unblocked shots and 8,572 goals. 2026-27 (`play_by_play_2027`) has 89 games through 2026-10-12. Shot-type values widen from 2022-23 (see below).

## Upstream source and delivery

- **Upstream.** NHL public API `api-web.nhle.com`, per-game `gamecenter/{id}/play-by-play`, plus shifts ([release notes](https://github.com/sportsdataverse/sportsdataverse-data/releases/tag/nhl_pbp_full)). This is not the legacy statsapi and not ESPN. sportsdataverse-py also has an ESPN pbp function (`espn_nhl_pbp`), which is a different, unrelated feed.
- **Delivery.** A **pre-built release**: parquet, csv and rds files on the GitHub release tag `nhl_pbp_full`, downloaded with no auth. `load_nhl_pbp(seasons)` / `load_nhl_pbp_full(seasons)` read `https://github.com/sportsdataverse/sportsdataverse-data/releases/download/nhl_pbp_full/play_by_play_{season}.parquet` and concatenate with `diagonal_relaxed`, because schemas drift by season. A lighter `nhl_pbp_lite` release holds the same data minus CHANGE rows ([`build.py` `pbp_lite`](https://github.com/sportsdataverse/fastRhockey-nhl-data/blob/ba888f6540a24dcf267eeadb96057b0a25c54617/python/nhl_data_build/build.py)).
- **Pipeline.**
  1. `fastRhockey-nhl-raw` captures per-game JSON (scheduled 08:00 UTC, Oct–Jun).
  2. `fastRhockey-nhl-data` builds and publishes (scheduled 09:00 UTC).

  The derived columns (`x_fixed`, `strength_state`, `empty_net`, block re-attribution, on-ice players) come from fastRhockey's R parser [`R/nhl_game_feed.R`](https://github.com/sportsdataverse/fastRhockey/blob/24653b5a502772db221e5d7081935c04ad414dfb/R/nhl_game_feed.R). The Python build ([`extract.py`](https://github.com/sportsdataverse/fastRhockey-nhl-data/blob/ba888f6540a24dcf267eeadb96057b0a25c54617/python/nhl_data_build/extract.py)) flattens the per-game result into season files.
- **Built-in `xg` column.** The release already carries an `xg` column from fastRhockey's own model. It is null on blocked shots and on about 2% of unblocked shots. It could serve as a public-model comparison. sportsdataverse-py has an open issue that the published boosters over-predict by 25–30% for seasons through 2023-24 ([sportsdataverse-py#444](https://github.com/sportsdataverse/sportsdataverse-py/issues/444)).

## Column schema (release, 94 columns in 2023-24+)

The full list is in the [loader docstring](https://github.com/sportsdataverse/sportsdataverse-py/blob/1ef5af272586e5d2c5873318e1f28d6a0d7bdbb3/sportsdataverse/nhl/nhl_loaders.py#L44-L178). Seasons before 2023-24 lack `game_date`, and before 2022-23 also lack `pptReplayUrl`. 2009-10 additionally lacks the on-ice, goalie and `strength_*` columns.

**Event types** (`event_type` / `event`), with 2025-26 counts:

| `event_type` | n | `event_type` | n |
|---|---|---|---|
| `CHANGE` | 656,504 | `TAKEAWAY` | 12,878 |
| `FACEOFF` | 78,491 | `PENALTY` | 10,496 |
| `SHOT` (on goal) | 69,507 | `GOAL` | 8,836 |
| `STOP` | 61,160 | `PERIOD_START` / `PERIOD_END` | 4,653 each |
| `HIT` | 59,279 | `DELAYED_PENALTY` (from 2019-20) | 3,643 |
| `BLOCKED_SHOT` | 43,664 | `GAME_END` | 1,394 |
| `GIVEAWAY` | 42,970 | `SHOOTOUT_COMPLETE` | 119 |
| `MISSED_SHOT` | 41,811 | `FAILED_SHOT_ATTEMPT` (shootout only) | 15 |

These map 1:1 to the raw `typeDescKey`s: `shot-on-goal`, `missed-shot`, `blocked-shot`, `goal`, and so on. For one game I fetched live, the counts match exactly. A Fenwick shot is `event_type ∈ {GOAL, SHOT, MISSED_SHOT}`.

**Shot type** is in `secondary_type`, lower-case:
- 2009-10 to 2021-22: `wrist, slap, snap, backhand, tip-in, deflected, wrap-around`.
- From 2022-23, additionally: `bat, poke, between-legs, cradle`. These are rare, about 0.6% of 2025-26 unblocked shots.
- Null on all `BLOCKED_SHOT` rows and on ≤ 0.5% of unblocked shots.
- For `PENALTY` rows, `secondary_type` holds the penalty `descKey`.

**Time.** Columns: `period` (int), `period_type` ∈ {`REGULAR`, `OVERTIME`, `SHOOTOUT`}, `period_time` (`"MM:SS"` elapsed), `period_seconds`, `period_seconds_remaining`, `game_seconds`, `game_seconds_remaining`. `game_seconds` is monotone within 99.9% of games. Regular-season shootouts are `period == 5`, `period_type == "SHOOTOUT"`. Playoff OT periods go up to 8 (2019-20).

**Teams and players.**
- Team: `event_team_abbr`, `event_team_type` (`home`/`away`), plus `home_abbr` and `away_abbr`.
- Players: `event_player_1_id/_name/_type` through `_3`.
  - Shots: player 1 is the `Shooter`, or the `Scorer` with assists after it.
  - Blocks: player 1 is the `Shooter` and player 2 the `Blocker`.
- Goalie: `event_goalie_id/_name`, null when the net is empty.

**Score.** `home_score`/`away_score` on a `GOAL` row already include that goal. A pre-shot score state must use the previous row's score.

**Coordinates and distance.** `x` and `y` are raw NHL feet: `x` runs from −99 to 99 and `y` from −42 to 42. `x_fixed`, `y_fixed`, `shot_distance` and `shot_angle` are derived. See the caveat section below.

**Identifiers.** `game_id` (e.g. `2025020001`), `season` (e.g. `"20252026"`), `season_type` (`R`/`P`), `game_date` (2023-24+), `event_idx`, `event_id`.

## Strength state, skaters, empty net, on-ice players

- **Skater counts.** `home_skaters`, `away_skaters`, `home_goalie_in` and `away_goalie_in` are parsed from the NHL `situationCode`. Its digits are away goalie, away skaters, home skaters, home goalie, so `"1551"` is 5v5 with both goalies in ([`.parse_situation_code`, L673](https://github.com/sportsdataverse/fastRhockey/blob/24653b5a502772db221e5d7081935c04ad414dfb/R/nhl_game_feed.R#L673)). Skater counts **include an extra attacker**, so a 6v5 with the goalie pulled shows as `6`.
- **`strength_state`.** Format `"{event-team skaters}v{opponent skaters}"`. For non-shootout Fenwick events it matches the event-team perspective in 100% of rows, and the home-away perspective in only 91%. 2025-26 Fenwick distribution: 5v5 128,860; 5v4 16,489; 4v5 7,557; 6v5 2,883; 5v6 2,169; 4v4 1,721; 3v3 1,607; `1v0` 835 (shootout and penalty shots); plus rarer states.
- **`strength_code` / `strength`.** Values are `EV`/`PP`/`SH`, but they are populated on only about 40% of events. Use `strength_state` or the skater counts instead.
- **`empty_net`.** TRUE when the event is an unblocked shot and the *opponent's* goalie is out (`*_goalie_in == 0`). It is always FALSE for blocks and non-shots ([`.extract_empty_net`, L763](https://github.com/sportsdataverse/fastRhockey/blob/24653b5a502772db221e5d7081935c04ad414dfb/R/nhl_game_feed.R#L763)). Empty-net shots appear as `5v6` or `4v6` from the shooter's view. In 2025-26 there were 1,068 empty-net unblocked shots, and on all of them `event_goalie_id` is null. Never null, populated in every season, including 2009-10.
- **`extra_attacker`.** TRUE when the event team's own goalie is out. It is also TRUE on penalty shots, because `1v0` situations encode the shooter's goalie as out.
- **On-ice players.** Columns: `home_on_1..7` / `away_on_1..7` (names), `*_on_*_id`, and `home_goalie` / `away_goalie` (+ `_id`). The `*_on_*` columns hold **skaters only**, and goalies sit in separate columns. The non-null `*_on_*_id` count equals the situationCode skater count on 98.3% of 2025-26 unblocked shots. About 0.5% of shots have no on-ice IDs at all.

## Penalty shots and shootouts

- **Shootout attempts.** `period_type == "SHOOTOUT"` with `strength_state == "1v0"`. In 2025-26 there were 264 goals, 394 shots and 139 misses, plus 15 `FAILED_SHOT_ATTEMPT` rows. There is also a separate `nhl_shootout` release.
- **Penalty shots.** There is **no explicit flag or text** ("penalty shot" never appears in `description`). They are identified two ways:
  - The shot row has `strength_state == "1v0"` while `period_type != "SHOOTOUT"`. That is 38 unblocked shots in 2025-26 and 46–47 in other seasons. It comes from raw `situationCode` `"1010"`/`"0101"`.
  - The preceding `PENALTY` row has a `secondary_type` beginning `ps-`, such as `ps-tripping-on-breakaway`, and `penalty_severity` null. Raw `details.typeCode == "PS"`.

  I verified game 2025020077 (P3 02:23) against the raw feed: a `penalty` with `typeCode` `"PS"` is followed by a `shot-on-goal` with `situationCode` `"1010"`.
- **Excluding both for an EW-style model:** drop `period_type == "SHOOTOUT"` and drop `strength_state == "1v0"`.

## Blocked shots: which team is the event team?

**In the release, the non-shooting (defending) team.** fastRhockey explicitly swaps the NHL `eventOwnerTeamId` for `BLOCKED_SHOT` rows ([L513, "Swap blocked shot team (blocker's team, not shooter's)"](https://github.com/sportsdataverse/fastRhockey/blob/24653b5a502772db221e5d7081935c04ad414dfb/R/nhl_game_feed.R#L513)). I measured this with on-ice IDs: in **every** season 2010-11 to 2026-27, the shooter (`event_player_1`) is **never** on `event_team` (0.000%). That includes `reason == "teammate-blocked"` rows (about 8% of blocks since 2023-24), where the blocker is actually the shooter's teammate.

**In the raw NHL feed the owner is the shooter's team.** In game 2026020087, two blocks have owner 13 = the shooter's team, with the blocker on team 7.

Consequences of the swap:
- `strength_state` for a block is from the defending team's view.
- `x_fixed` puts the block in the defending team's frame, mirrored relative to the shot.
- `shot_distance` for a block is still the distance to the net being shot at, because it is computed on `abs(x)`.

For prior-event features ("same team as shooter?"), a block's shooting team is `event_player_1`'s team, which is the *opposite* of `event_team_abbr`.

## Coordinates: what `x_fixed` really is

The docs describe `x_fixed` as "Normalized x coordinate (home shoots right)". In the source, every `homeTeamDefendingSide` branch resolves to `x_fixed = abs(x)` for the home team and `-abs(x)` for the away team ([`.fix_coordinates`, L789](https://github.com/sportsdataverse/fastRhockey/blob/24653b5a502772db221e5d7081935c04ad414dfb/R/nhl_game_feed.R#L789)). Measured on 2025-26 Fenwick rows, `x_fixed == sign(team) * abs(x)` in 100% of rows. `y_fixed` is negated only for events whose raw `x` lies on the "wrong" half, so for ordinary attacking-zone shots `y_fixed == y`. The rink is mirrored, not rotated: left/right side is not consistent, and `shot_angle` is unsigned. `shot_distance` is `sqrt((89 − |x|)² + y²)` and `shot_angle` is the absolute angle to (±89, 0), with behind-the-net angles > 90 ([`.add_shot_metrics`, L860](https://github.com/sportsdataverse/fastRhockey/blob/24653b5a502772db221e5d7081935c04ad414dfb/R/nhl_game_feed.R#L860)).

What this means in practice:
- **Shots in the attacking half: correct.** Distance and |angle| are right.
- **Shots from beyond centre: distorted.** `abs(x)` folds them into the attacking half, so the published distance is far too short. Example: an empty-net miss at raw x = −86 for a team attacking +x gets `shot_distance` 28.2.
- **Size of the problem.** About 2.3% of 2025-26 unblocked shots (1.0% in 2011–16, 2.0–2.4% from 2016-17) are on the minority side for their team and period.
  - Their goal rate is 6.5%, versus 7.2% for the rest. That points to most being NHL-feed side errors, which the `abs()` actually repairs, rather than genuine 150-ft shots.
  - But 19% of them are empty-net shots, which genuinely are long-range, and those come out wrong.
  - This is a heuristic check: majority side per game, period and team. The release does not carry `homeTeamDefendingSide`, so there is no ground truth in the file.
- **The raw feed has the true side.** It carries `homeTeamDefendingSide` per play ("left"/"right"), so a parser working from raw JSON can normalise correctly.

## CHANGE rows have `game_id = null`

In every season with shifts (2010-11 on), all CHANGE rows have `game_id` null, and also null `event_team_abbr`. In 2025-26 that is 656,504 rows, so a `group_by("game_id")` puts every shift row into a single null group. That is why "1,394 of 1,395 games" appear to have no shift rows: the 1,395th "game" is the null group, which holds all the shifts.

The rows themselves are intact:
- `season`, `period`, `game_seconds`, `players_on`/`ids_on`, `home_abbr` and `event_idx` are populated.
- They sit physically inside their game's block of rows. 99.6% fall strictly between that game's first and last event rows, where forward-fill and backward-fill of `game_id` agree.
- That gives about 471 CHANGE rows per game, min 371.

A likely cause is the build's `_attach_ids`, which fills `game_id` with `dict.setdefault`. If CHANGE rows arrive with an explicit `game_id: null`, `setdefault` won't overwrite it ([`extract.py` L26–33](https://github.com/sportsdataverse/fastRhockey-nhl-data/blob/ba888f6540a24dcf267eeadb96057b0a25c54617/python/nhl_data_build/extract.py#L26-L33)). This is an inference; I did not inspect the intermediate JSON.

What it affects:
- **On-ice players and strength state on event rows: no.** They are computed per game before season assembly and come from `situationCode`, not CHANGE rows. The 98.3% on-ice-count agreement above was measured on event rows.
- **Prior-event construction: yes, if CHANGE rows are kept.** A `shift().over("game_id")` would break. EW-style prior events don't use line changes, so drop CHANGE rows first (or load `nhl_pbp_lite`). After that, the prior event of an unblocked shot is one of: FACEOFF, SHOT, HIT, GIVEAWAY, BLOCKED_SHOT, MISSED_SHOT, TAKEAWAY, PERIOD_START, DELAYED_PENALTY, GOAL, PERIOD_END.
- **Shift-based features: yes.** Time-on-ice, or "seconds since last change", would need a fill of `game_id` within contiguous blocks first, for example backward-fill in file order.

## Freshness: can it score this week's games?

- **The release is daily.** The `play_by_play_2027` asset and `timestamp.json` were last written `2026-10-13 04:01:30 EDT` (08:01 UTC). The scheduled jobs are raw scrape at 08:00 UTC and build at 09:00 UTC, Oct–Jun ([release notes](https://github.com/sportsdataverse/sportsdataverse-data/releases/tag/nhl_pbp_full)). On 2026-10-13 it held **all 89** regular-season games that were final on the NHL API through 2026-10-12, with none missing. Latency is next morning; games in progress are never included.
- **Same-day games need a live path, which returns raw data.** `sportsdataverse.nhl.nhl_web_pbp(game_id)` fetches `api-web.nhle.com/v1/gamecenter/{id}/play-by-play` live ([`nhl_api_web.py` L66](https://github.com/sportsdataverse/sportsdataverse-py/blob/1ef5af272586e5d2c5873318e1f28d6a0d7bdbb3/sportsdataverse/nhl/nhl_api_web.py#L66)). But its parser only flattens the raw plays into 49 columns of snake_case raw fields (`situation_code`, `home_team_defending_side`, `details_x_coord`, `details_event_owner_team_id`, `details_shot_type`, …). There is no `strength_state`, on-ice players, `x_fixed`, block swap or names. The release schema is only produced by the R `fastRhockey::nhl_game_pbp`. Scoring a game before the next release therefore needs the project's own raw-to-features parser. Done well, that parser would also fix the `x_fixed` and block issues.

## Install and maintenance

- **PyPI.** `sportsdataverse` 0.1.4 (released 2026-09-01), `requires_python >= 3.9`, MIT licence ([PyPI](https://pypi.org/project/sportsdataverse/)). Releases are frequent: 0.0.73 → 0.1.4 between 2026-08-01 and 2026-09-01, and docs for 0.1.5 are already in the repo.
- **Python 3.14 on Windows: works.** In a throwaway `uv init --python 3.14` project, `uv add sportsdataverse` resolved and installed from wheels (polars 1.44, pandas 3.0, pyarrow 26, xgboost 3.4, scikit-learn 1.9). `load_nhl_pbp(seasons=[2027])` (70,438 × 94, 89 games), `nhl_web_pbp` and `nhl_score` all ran under 3.14.8.
- **Weight: heavy.** The package pulls matplotlib, scikit-learn, xgboost, scipy, pyjanitor, beautifulsoup and lxml. The project only needs the parquet URLs, which `polars.read_parquet` can read directly without the package.
- **Activity: maintained.** sportsdataverse-py, fastRhockey-nhl-data and the release were all pushed on 2026-10-13.
- **Open issues relevant here:**
  - [fastRhockey-nhl-data#16](https://github.com/sportsdataverse/fastRhockey-nhl-data/issues/16): season-file mislabelling.
  - [fastRhockey-nhl-data#17](https://github.com/sportsdataverse/fastRhockey-nhl-data/issues/17): `nhl_schedule_2026.parquet` R metadata makes polars panic. That affects the schedule release, not pbp.
  - [sportsdataverse-py#444](https://github.com/sportsdataverse/sportsdataverse-py/issues/444): the built-in xG model over-predicts.

## Implications for the plan (summary)

| Threat | Severity | Handling |
|---|---|---|
| 2014-15 missing, 2015–2018 files off by one, 2018-19 duplicated | High (silent) | Key seasons on the `season` column, dedupe on `game_id`, accept or backfill the 2014-15 gap |
| `x_fixed` sign-forced, no `homeTeamDefendingSide` | Medium | Fine for attacking-half distance and |angle|. Long empty-net shots are mis-distanced. Infer the side per game-period from raw `x`, or parse raw JSON |
| Blocked-shot event team = defending team | Medium | Derive the shooting team from `event_player_1` for prior-event team features |
| CHANGE rows have null `game_id` | Low | Drop CHANGE rows (or use `nhl_pbp_lite`) before per-game lags |
| 2019-20: 174 games without strength state | Low–Medium | Drop them or rebuild their strength state from shifts |
| 2009-10 has no strength or on-ice columns | Low | Start training at 2010-11 or later |
| Score columns are post-event on GOAL rows | Low | Lag the score for the score-state feature |
| Daily release only; live function is raw schema | Medium | Own raw-to-features parser if same-day scoring matters; otherwise next-morning release is fine |
| Penalty shots have no explicit flag | Low | `strength_state == "1v0"` outside the shootout |
| Python 3.14 | None | Installs and runs |
