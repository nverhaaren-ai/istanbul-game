# Design: LLM Player Support

> **Executive summary:** This plan introduces LLM player support through a layered
> architecture across four phases. Phase 0 replaces the engine's compound action types
> (which fuse player intent with random outcomes) with a sub-step model: actions that
> involve dice or card draws are decomposed into sequential steps, each taken by the
> appropriate actor — either the current player or a configurable randomness resolver.
> This also cleanly supersedes the police station temporary-relocation hack. Phases 1–2
> add a pure `is_valid_action()` predicate and `legal_actions()` API on top of the
> refactored engine. Phase 3 adds a separate JSON interface module exposing the engine
> state and actions in a tool-call format suitable for both LLM and human interfaces.
> No open design questions remain before implementation can begin.

---

## Goals

- Decompose compound actions into discrete per-actor sub-steps, cleanly separating
  player decisions from random outcomes.
- `GameState.is_valid_action(action) -> bool` — pure predicate, never raises.
- `GameState.legal_actions() -> list[PlayerAction]` — exact set of legal actions for
  the current actor at this moment.
- A separate `istanbul_game/interface.py` module that translates engine state and actions
  into a JSON tool-call representation, usable by both LLM and human interfaces.

---

## Architectural Layers

```
┌─────────────────────────────────┐
│   Runner / LLM integration      │  drives the game loop, routes to player/resolver
├─────────────────────────────────┤
│   interface.py                  │  JSON ↔ Python action/state translation
├─────────────────────────────────┤
│   GameState / TurnState         │  engine: validation, state mutation, sub-step tracking
│   actions.py (revised)          │  decomposed action types
└─────────────────────────────────┘
```

The engine is unaware of JSON. The interface module is unaware of the runner. The runner
knows both.

---

## Phase 0: Sub-step Engine Model

### Problem with compound actions

Several existing action types fuse a player's intent with a random outcome in a single
object: `TeaHouseAction(call, roll)`, `BlackMarketAction(good, roll)`,
`EncounterGovernor(gain, cost, roll)`, `EncounterSmuggler(gain, cost, roll)`,
`MarketAction(goods, new_demand)`. This makes it impossible for the player to make a
decision *after* seeing the random outcome (most critically, whether to use the red tile),
and forces awkward sentinel-value workarounds at the interface layer.

`PoliceStationAction(location, sub_action)` has a related problem: it embeds a nested
action and the handler resorts to temporarily relocating the player so sub-handlers see
the right tile. Both issues share the same root cause — the engine has no concept of
"the current step within a turn."

### Solution: sub-step tracking in TurnState

Extend `TurnState` with three new fields:

| Field | Type | Purpose |
|-------|------|---------|
| `current_actor` | `ActorType` | `PLAYER` or `RESOLVER` |
| `pending_sub_step` | `SubStepType \| None` | What the current actor must provide next |
| `dispatched_location` | `Location \| None` | Effective tile location during a police station dispatch |

`pending_sub_step` is one of: `AWAITING_DICE_ROLL`, `AWAITING_SECOND_DICE_ROLL`,
`AWAITING_RED_TILE_DECISION`, `AWAITING_DEMAND_DRAW`.

When `pending_sub_step` is not None, `legal_actions()` returns only the actions
appropriate to that sub-step (resolver or player depending on `current_actor`). Phase
advancement does not happen until the sub-step sequence completes.

### New and revised action types

**Decomposed player intent actions** (replace the compound types):

| New action | Replaces | Notes |
|-----------|----------|-------|
| `TeaHouseCall(call: int)` | `TeaHouseAction` | Player half only |
| `BlackMarketGoodChoice(good: Good)` | `BlackMarketAction` | Player half only |
| `MarketGoodsChoice(goods: Counter[Good])` | `MarketAction` | Player half only; no `new_demand` |
| `SendFamilyMember(location: Location)` | `PoliceStationAction` | Sub-action handled separately via dispatch context |
| `EncounterGovernorIntent(gain: Card, cost: Card \| Pay)` | `EncounterGovernor` | Player half only |
| `EncounterSmugglerIntent(gain: Good, cost: Good \| Pay)` | `EncounterSmuggler` | Player half only |
| `RedTileDecision(use: bool, method: str \| None, die_index: int \| None)` | *(new)* | Player follow-up after seeing dice |

**Resolver actions** (new; taken by the randomness resolver, not the player):

| Action | Triggers when |
|--------|--------------|
| `DiceRoll(result: tuple[int, int])` | `pending_sub_step` is `AWAITING_DICE_ROLL` or `AWAITING_SECOND_DICE_ROLL` |
| `DemandDraw(demand: Counter[Good])` | `pending_sub_step` is `AWAITING_DEMAND_DRAW` |

**Unchanged action types:** all movement actions, `Pay`, `YieldTurn`, `ChooseReward`,
`SultansPalaceAction`, `MosqueAction`, `FountainAction`, `GenericTileAction`,
`CaravansaryAction`, `SkipTileAction`, and all card actions (`OneGood`, `FiveLira`,
`ArrestFamily`, `YellowTile`, `GreenTile`, `SellAny`, `DoubleCard`).

### Sub-step sequences

**Tea House:**
1. `TeaHouseCall(call)` [PLAYER] → `pending_sub_step = AWAITING_DICE_ROLL`, `current_actor = RESOLVER`
2. `DiceRoll(result)` [RESOLVER] →
   - If player has red tile: `pending_sub_step = AWAITING_RED_TILE_DECISION`, `current_actor = PLAYER`
   - Otherwise: compute outcome (lira gain), proceed to phase 4
3. `RedTileDecision(use, ...)` [PLAYER] →
   - `use=False`: compute outcome, proceed to phase 4
   - `use=True, method=TO_FOUR`: apply deterministic die modification, compute outcome, proceed to phase 4
   - `use=True, method=REROLL`: `pending_sub_step = AWAITING_SECOND_DICE_ROLL`, `current_actor = RESOLVER`
4. `DiceRoll(result)` [RESOLVER] (only on REROLL) → compute outcome, proceed to phase 4

**Black Market:** identical pattern to Tea House with `BlackMarketGoodChoice` at step 1.

**Market:**
1. `MarketGoodsChoice(goods)` [PLAYER] → `pending_sub_step = AWAITING_DEMAND_DRAW`, `current_actor = RESOLVER`
2. `DemandDraw(demand)` [RESOLVER] → apply trade and set new demand, proceed to phase 4

**Governor / Smuggler encounters:**
1. `EncounterGovernorIntent(gain, cost)` [PLAYER] → `pending_sub_step = AWAITING_DICE_ROLL`, `current_actor = RESOLVER`
2. `DiceRoll(result)` [RESOLVER] → move NPC to rolled location, complete encounter
(No red tile involvement; the roll determines NPC movement only.)

**Police station:**
1. `SendFamilyMember(location)` [PLAYER] → validate preconditions, move family member,
   set `dispatched_location = location`, remain in phase 3
2. Player takes the appropriate tile action at `dispatched_location` (including any
   sub-steps for that tile, e.g., if the destination is the Tea House)
3. After the tile action and all its sub-steps complete, `dispatched_location` is cleared
   and the turn proceeds to phase 4

`dispatched_location` replaces the `family_member_acting` flag and the temporary player
relocation entirely. Handlers that currently read `player_state.location` to determine
the operating tile instead read `turn_state.dispatched_location or player_state.location`.

### Randomness resolver

The resolver is a runner-layer concept, not an engine concept. The engine accepts
`DiceRoll` and `DemandDraw` actions via `take_action()` when `current_actor == RESOLVER`;
it does not care who generated them.

```python
class RandomnessResolver(Protocol):
    def resolve_dice(self) -> tuple[int, int]: ...
    def resolve_demand(self) -> Counter[Good]: ...
```

Three implementations:
- `ProgramResolver` — generates random values; **default**, most convenient for physical play
- `HumanResolver` — prompts the operator for each value (physical dice, deck draws)
- `ReplayResolver` — reads pre-recorded values from a sequence (for replaying old games)

### Replay format migration

The new action sequences are not backward-compatible with the existing CSV/JSON replay
format, which records compound actions atomically. A conversion shim can be written when
needed: each old compound action maps to a deterministic sequence of new actions:

| Old action | New sequence |
|-----------|-------------|
| `TeaHouseAction(call, (a,b))` | `TeaHouseCall(call)`, `DiceRoll((a,b))`, `RedTileDecision(use=False)` |
| `TeaHouseAction(call, RedTileAction(i, f, TO_FOUR))` | `TeaHouseCall(call)`, `DiceRoll(i)`, `RedTileDecision(use=True, TO_FOUR, ...)` |
| `TeaHouseAction(call, RedTileAction(i, f, REROLL))` | `TeaHouseCall(call)`, `DiceRoll(i)`, `RedTileDecision(use=True, REROLL)`, `DiceRoll(f)` |
| `BlackMarketAction(good, roll)` | analogous to Tea House |
| `MarketAction(goods, demand)` | `MarketGoodsChoice(goods)`, `DemandDraw(demand)` |
| `EncounterGovernor(gain, cost, roll)` | `EncounterGovernorIntent(gain, cost)`, `DiceRoll(roll)` |
| `EncounterSmuggler(gain, cost, roll)` | `EncounterSmugglerIntent(gain, cost)`, `DiceRoll(roll)` |
| `PoliceStationAction(loc, sub)` | `SendFamilyMember(loc)`, then converted `sub` |

The shim does not need to be built now; it can be deferred until there are old replays
that need to be preserved.

---

## Phase 1: Pure Validator — `is_valid_action`

Add `GameState.is_valid_action(action: PlayerAction) -> bool` as a pure predicate.
It never raises and never mutates state. `take_action()` keeps its existing
assert-on-invalid behaviour but gates on `is_valid_action()`.

`legal_actions()` returns actions for `current_actor`: player actions when
`current_actor == PLAYER`, resolver actions when `current_actor == RESOLVER`.
The validator enforces this: a `DiceRoll` is invalid when `current_actor == PLAYER`
and vice versa.

### Preconditions by action type

**Cross-cutting:**
- `ChooseReward`: `outstanding_reward_choices > 0`; valid choice value
- `YieldTurn`: phase in {2, 4}; `outstanding_reward_choices == 0`; `pending_sub_step is None`
- All player actions: `current_actor == PLAYER`

**Phase 1:**
- `Move(loc, skip)`: `taxicab_dist(current, loc) in {1, 2}`
- `ExtraMoveCardAction(move)`: has EXTRA_MOVE card; `taxicab_dist(current, move.tile) in {3, 4}`
- `NoMoveCardAction`: has NO_MOVE card
- `ReturnAssistantCardAction(from_tile)`: has RETURN_ASSISTANT card; assistant at `from_tile`
- `OneGoodCardAction`, `FiveLiraCardAction`, `ArrestFamilyCardAction`, `YellowTileAction`: card/tile held, other standard preconditions

**Phase 2:** `Pay`: other players at tile; lira >= (count − 1) × 2

**Phase 3 tile actions** (effective tile = `dispatched_location or current_player_location`):

| Action | Key preconditions |
|--------|------------------|
| `GenericTileAction` | at correct tile for that action; resource checks (lira, extensions) as applicable |
| `FountainAction(locs)` | at Fountain; `locs ⊆ assistant_locations`; non-empty |
| `MosqueAction(good)` | at mosque; player lacks tile; mosque offers it; has goods to pay |
| `BlackMarketGoodChoice(good)` | at Black Market |
| `CaravansaryAction(gains, cost)` | at Caravansary; has cost card; discard depth ≥ DISCARD count; not awaiting_discard |
| `MarketGoodsChoice(goods)` | at Small or Large Market; has those goods; within demand; not expecting_demand |
| `TeaHouseCall(call)` | at Tea House; call in {3..12} |
| `SultansPalaceAction(goods)` | at Sultan's Palace; `required()` not None; goods satisfies requirement |
| `SendFamilyMember(location)` | at Police Station; player has family there; location ≠ police station |
| `SkipTileAction` | always valid in phase 3 |
| `GreenTileAction(good)` | has GREEN tile; at warehouse; lira >= 2 |
| `SellAnyCardAction(action)` | has SELL_ANY card; at SMALL_MARKET; inner action valid |
| `DoubleCardAction(card, actions)` | has card; at correct tile; each sub-action valid |

**Phase 4:**
- `EncounterGovernorIntent(gain, cost)`: governor at tile; can pay cost
- `EncounterSmugglerIntent(gain, cost)`: smuggler at tile; can pay cost

**Resolver actions:**
- `DiceRoll(result)`: `current_actor == RESOLVER`; `pending_sub_step in {AWAITING_DICE_ROLL, AWAITING_SECOND_DICE_ROLL}`; result is a valid 2d6 tuple
- `DemandDraw(demand)`: `current_actor == RESOLVER`; `pending_sub_step == AWAITING_DEMAND_DRAW`; demand sums to 5

**Sub-step actions:**
- `RedTileDecision(use=False)`: `pending_sub_step == AWAITING_RED_TILE_DECISION`
- `RedTileDecision(use=True, ...)`: additionally, player has red tile; modification is valid for the recorded initial roll

### Implementation note

`_can_do_tile_action_at(action, location) -> bool` is a private helper performing phase 3
validation at an arbitrary location — used by the main validator (passing
`dispatched_location or current_player_location`) and directly for validating tile actions
during a police station dispatch without any player relocation.

---

## Phase 2: Candidate Generators

`legal_actions()` = filter(`is_valid_action`, `_candidate_actions()`).

Because player actions no longer contain random outcomes, no sentinel values appear in
the player action candidates. Resolver action candidates (`DiceRoll`, `DemandDraw`) are
generated separately when `current_actor == RESOLVER`.

### By phase / sub-step

**Phase 1:** `Move`, `ExtraMoveCardAction`, `NoMoveCardAction`, `ReturnAssistantCardAction`
for all applicable locations; phase-all cards.

**Phase 2:** `YieldTurn`, `Pay`, phase-all cards.

**Phase 3 tile candidates:**

| Tile | Candidates |
|------|-----------|
| POST_OFFICE, warehouses, WAINWRIGHT, GEMSTONE_DEALER | `GenericTileAction()` |
| FOUNTAIN | `GenericTileAction()` (recall all); `FountainAction(subset)` for each non-empty proper subset of `assistant_locations` (≤ 31) |
| GREAT / SMALL MOSQUE | `MosqueAction(good)` for each available good |
| BLACK_MARKET | `BlackMarketGoodChoice(good)` for each of {RED, GREEN, YELLOW} |
| CARAVANSARY | `CaravansaryAction(gains, cost)` for each card in hand × valid gain combination |
| SMALL / LARGE MARKET | `MarketGoodsChoice(goods)` for each non-empty subset of (cart ∩ demand) |
| TEA_HOUSE | `TeaHouseCall(call)` for call in {3..12} |
| SULTANS_PALACE | `SultansPalaceAction(goods)` for each valid `Counter[Good]` satisfying `required()` |
| POLICE_STATION | `SendFamilyMember(location)` for each non-police-station location |

**Phase 3 card candidates:** `GreenTileAction`, `SellAnyCardAction`, `DoubleCardAction`
as before.

**Phase 4:** `EncounterGovernorIntent(gain, cost)` for each Card gain × each (Card in
hand + Pay) as cost; analogously for smuggler; `ChooseReward` if outstanding; `YieldTurn`.

**Sub-step candidates:**
- `AWAITING_DICE_ROLL` / `AWAITING_SECOND_DICE_ROLL`: `DiceRoll(result)` for all valid
  2d6 results (for testing/human input) — in practice the resolver generates one value
- `AWAITING_RED_TILE_DECISION`: `RedTileDecision(use=False)`; if player has red tile,
  also `RedTileDecision(use=True, TO_FOUR, die=0)`, `RedTileDecision(use=True, TO_FOUR, die=1)`,
  `RedTileDecision(use=True, REROLL)`
- `AWAITING_DEMAND_DRAW`: `DemandDraw(demand)` for any valid demand counter

**Sultan's Palace goods:** Enumerate all `Counter[Good]` satisfying `required()` given
cart contents — tractable with 4 good types and slowly-growing wildcard count.

---

## Phase 3: JSON Interface Module

### Module: `istanbul_game/interface.py`

A pure translation layer with no game logic. Takes Python action/state objects from the
engine; returns JSON-serialisable dicts. Has no imports from the runner layer.

Primary functions:

```python
def legal_actions_json(actions: list[PlayerAction]) -> list[dict]: ...
def game_state_json(gs: GameState) -> dict: ...
def action_from_json(data: dict) -> PlayerAction: ...
def action_guide_markdown() -> str: ...
```

### Action representation — tool-call format

Each legal action is a JSON object with `name` and `args`:

```json
{"name": "TeaHouseCall", "args": {"call": 7}}
{"name": "MarketGoodsChoice", "args": {"goods": {"red": 1, "green": 2}}}
{"name": "SendFamilyMember", "args": {"location": 5}}
{"name": "DiceRoll", "args": {"result": [3, 4]}}
```

When multiple actions have the same `name` but differ only in one independent parameter
dimension, the interface module **may** collapse them into a Cartesian product form if
it reduces noise without losing information:

```json
{
  "name": "TeaHouseCall",
  "args": {"call": {"type": "int", "range": [3, 12]}}
}
```

This is a display optimisation; `action_from_json` always accepts both the explicit and
collapsed forms. Whether to collapse a given action type is decided per-type at
implementation time. Good candidates: `TeaHouseCall` (call range), `Move` (destination
grid), `EncounterGovernorIntent` (gain × cost grid). Poor candidates: `MarketGoodsChoice`
(interdependent goods constraints), `SultansPalaceAction` (goods constrained by cart).

### Game state description — `game_state_json()`

```
current_player:
  color, phase, pending_sub_step, current_actor
  lira, rubies
  cart: {red, blue, green, yellow, cart_max}
  hand: {card_name: count, ...}
  assistant_locations: [location_numbers]
  family_location: location_number

other_players: [{color, location, rubies}, ...]   # public info only

board:
  governor_location, smuggler_location
  tile_layout: {location_number: tile_name, ...}

tiles:
  small_market: {demand: {good: count}, one_cost: int}
  large_market: {demand: {good: count}, one_cost: int}
  post_office: {position: int, next_goods: [good_names], next_lira: int}
  great_mosque: {available_tiles: {good: cost}}
  small_mosque: {available_tiles: {good: cost}}
  sultans_palace: {required_count: int, required: {good_or_any: count}}
  gemstone_dealer: {cost: int | null}
  wainwright: {extensions_remaining: int}
  caravansary: {discard_pile_depth: int, discard_pile_top: card_name | null}

outstanding_reward_choices: int
```

### Action guide

`action_guide_markdown()` returns a static markdown document (suitable for inclusion in
an LLM system prompt) explaining:

- The turn structure (phases, sub-steps)
- Each action name, its arguments, and when it is available
- What resolver actions are (`DiceRoll`, `DemandDraw`) and that the runner provides them
  automatically when using `ProgramResolver`
- How `RedTileDecision` works and when it appears
- How to request per-action documentation or evaluate a specific action call

The LLM may also request a full explicit enumeration of legal actions if the compact
form is ambiguous — the runner should support a re-query that provides the expanded list.

---

## Testing

- **Phase 0:** All existing tests pass unchanged (observable behaviour is identical).
  New tests cover each sub-step sequence end-to-end (Tea House, Black Market, Market,
  Governor, Smuggler, Police Station dispatch) using each resolver type.
- **Phase 1:** Unit tests for `is_valid_action` covering valid and invalid cases for
  each action type and sub-step state. Property test: no action returned by
  `legal_actions()` raises when passed to `take_action()`, and no action that does not
  raise in `take_action()` is absent from `legal_actions()`.
- **Phase 2:** `legal_actions()` tested against known game states; no legal action
  omitted, no illegal action included. Separate tests for resolver action candidates.
- **Phase 3:** `game_state_json()` and `legal_actions_json()` snapshot tests.
  Round-trip test: `action_from_json(legal_actions_json([a])[0]) == a` for all action types.

---

## File layout

| File | Contents |
|------|----------|
| `istanbul_game/actions.py` | New decomposed action types alongside retained types; old compound types removed or kept in `actions_legacy.py` for shim use |
| `istanbul_game/turn.py` | `TurnState` extended with `current_actor`, `pending_sub_step`, `dispatched_location` |
| `istanbul_game/game.py` | `is_valid_action`, `_can_do_tile_action_at`, `_candidate_actions`, `legal_actions`; refactored handlers reading from execution context |
| `istanbul_game/resolver.py` | `RandomnessResolver` protocol; `ProgramResolver`, `HumanResolver`, `ReplayResolver` |
| `istanbul_game/interface.py` | `legal_actions_json`, `game_state_json`, `action_from_json`, `action_guide_markdown` |
| `istanbul_game/candidates.py` | Candidate generator helpers if `game.py` becomes unwieldy |
| `istanbul_game/shim.py` | Replay format conversion (deferred; implement when needed) |
| `tests/test_substeps.py` | Sub-step sequence tests |
| `tests/test_validator.py` | `is_valid_action` unit tests |
| `tests/test_legal_actions.py` | `legal_actions()` integration tests |
| `tests/test_interface.py` | JSON round-trip and snapshot tests |
