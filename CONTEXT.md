# NHL Expected Goals

An expected goals model for NHL shots, modelled on Evolving-Wild's 2018 write-up: the probability that a given unblocked shot becomes a goal, rolled up for player and team evaluation.

## Language

### Events and shots

**Play-by-play**:
An event-by-event record of a game: each row is one event with its type, team, players, coordinates, clock and on-ice context. Usually the NHL's own, but it may come from another source or league.
_Avoid_: PBP data, RTSS (the old name for the NHL feed)

**Scoreable game**:
A game whose play-by-play exists, whether finished or still in progress.
_Avoid_: Available game, complete game

**Fenwick shot**:
An unblocked shot attempt: a goal, a shot on goal, or a missed shot. The unit the model scores.
_Avoid_: Shot attempt (that includes blocked shots), unblocked attempt

**Prior event**:
The play-by-play event immediately before a Fenwick shot within the same game period, used to describe how the shot came about.
_Avoid_: Last event, previous play

**Strength state**:
The manpower situation at a shot, from the shooter's side: how many skaters each team has on the ice and whether the opponent's net is empty.
_Avoid_: Game state, situation, manpower

**Extra attacker**:
A skater on the ice in place of their team's pulled goalie; counts as a skater in the strength state.
_Avoid_: Sixth attacker

**Excluded shot**:
A Fenwick shot that no strength-state model scores (e.g. a penalty shot, or one missing coordinates); it gets no xG and carries the reason it was excluded.
_Avoid_: Dropped shot, filtered shot

### Models and outputs

**Expected goals (xG)**:
The modelled probability that a Fenwick shot is a goal; summed over shots to give a player's or team's expected goals.
_Avoid_: Shot quality, goal probability

**Provisional xG**:
Expected goals for a game the league has not yet made official; it can still change until the game is official.
_Avoid_: Live xG, partial xG

**Strength-state model**:
One of four separate models, each scoring the Fenwick shots from one group of strength states: empty net takes every shot at an empty net; the rest are split by skater difference into even strength, power play and shorthanded.
_Avoid_: Sub-model, situational model

**Training window**:
The run of most recent complete seasons a strength-state model is trained on; its length is set per model.
_Avoid_: Lookback, training range

### Validation

**Test season**:
A complete season kept out of a model's training so the model can be judged on games it has never seen.
_Avoid_: Holdout season, validation season

**Season-forward validation**:
Judging a strength-state model by training it only on seasons before a test season, then scoring that test season; repeated over several recent test seasons. Mirrors how a frozen model meets a new season.
_Avoid_: Time-series CV, rolling CV, backtest

**Calibration**:
How closely expected goals match actual goals over a group of Fenwick shots: well calibrated means the summed xG lands near the goals scored, across the whole range of predicted probabilities.
_Avoid_: Accuracy, bias

**Calibration gate**:
The pass/fail calibration test a strength-state model must clear before it ships.
_Avoid_: Acceptance test, quality bar

**Trip-wire**:
A check that never fails the build on its own but, when crossed, means something must be investigated before shipping (e.g. a suspiciously high AUC pointing at leakage).
_Avoid_: Soft gate, warning
