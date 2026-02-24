# Design: LLM Player Support

> **Executive summary:** This plan introduces LLM player support for the Istanbul game
> engine through three sequential phases: a prerequisite refactor to eliminate the police
> station temporary-relocation hack, extraction of a pure `is_valid_action()` predicate,
> and implementation of candidate generators feeding a `legal_actions()` API plus a
> `describe()` state snapshot. Sentinel values are used only for genuinely random
> externally-resolved parameters (dice rolls, demand card draws); all other parameters
> are enumerated in full. One open design question must be resolved before Phase 2
> implementation: the multi-step interaction protocol for dice-dependent red tile
> decisions at the Tea House and Black Market (see "Dice and red tile sequencing" below).

This document describes the plan for enabling an LLM to act as one or more players
in the Istanbul game engine. The approach is to expose a `legal_actions()` API that
returns the exact set of valid actions at any point in the game, plus a game-state
description suitable for LLM consumption.

## Goals

- `GameState.legal_actions() -> list[PlayerAction]` — returns exactly the legal actions
  available to the current player at this moment (no illegal actions included, no legal
  actions omitted).
- `GameState.is_valid_action(action) -> bool` — pure predicate, never raises, replaces
  assertion-based validation as the source of truth. `take_action()` continues to assert
  on this, but callers (including the LLM interface) can check without crashing.
- Clean up the police station temporary-relocation hack as a prerequisite, since it
  makes the validator structurally awkward.

## Phase 0: Refactor Police Station Temporary Relocation

### Problem

`_handle_police_station_action` (game.py:518–542) temporarily overwrites
`player_state.location` and the tile's `players` sets so that recursive sub-handlers see
the player as being at the destination tile. A comment acknowledges this is an oversight.
This pattern makes it impossible to validate a `PoliceStationAction`'s sub-action without
actually relocating, and it makes handlers harder to test in isolation.

### Solution: `_execute_tile_action_at(action, location)`

Extract a dispatcher method that routes a tile action to the correct handler, using
an explicit `location` argument rather than reading `player_state.location`. Both the
main `take_action()` and the police station handler call this method. Handlers are
refactored to receive the resolved `tile` and `tile_state` as parameters instead of
computing them from player location.

```
_execute_tile_action_at(action: PlaceTileAction | GreenTileAction, location: Location)
    tile = location_map[location]
    tile_state = tile_states[tile]
    dispatch to the appropriate handler, passing tile and tile_state explicitly
```

**Changes to handlers:** Each `_handle_X` method receives `tile: Tile` and
`tile_state: TileState` as parameters, dropping the `self.location_map[player_state.location]`
call. The tile-identity assertions (`assert tile is Tile.FOO`) become internal sanity checks
rather than the primary validation mechanism.

**Police station handler becomes:**

```python
def _handle_police_station_action(self, action: PoliceStationAction) -> None:
    # validate and move family member ...
    self.family_member_acting = True
    self._execute_tile_action_at(action.action, action.location)
    self.family_member_acting = False
    # no player relocation needed
```

**Main take_action phase 3 dispatch becomes:**

```python
# instead of the current generic_action_map + match block:
self._execute_tile_action_at(action, self.current_player_location)
self._encounter_family_members()
```

This phase 0 refactor should be covered by existing tests (behaviour is unchanged).

---

## Phase 1: Pure Validator — `is_valid_action`

### Structure

Add `GameState.is_valid_action(action: PlayerAction) -> bool` as a pure predicate.
It never raises and never mutates state. `take_action()` keeps its existing behaviour
(assert-on-invalid) but gates on `is_valid_action()` rather than inline assertions.

The validator has two layers, mirroring the existing two-level validation:

1. **Phase/type check** — delegates to `TurnState.valid_action()` (already exists).
2. **State check** — for each action type, check the preconditions that the handler
   currently asserts.

### Preconditions by action type

**Cross-cutting:**
- `ChooseReward`: `outstanding_reward_choices > 0`; choice is LIRA or a valid Card
- `YieldTurn`: phase in {2, 4} and `outstanding_reward_choices == 0`
- All others: `not completed`

**Phase 1 actions:**
- `Move(loc, skip)`: `taxicab_dist(current, loc) in {1, 2}`
- `ExtraMoveCardAction(move)`: has EXTRA_MOVE card; `taxicab_dist(current, move.tile) in {3, 4}`
- `NoMoveCardAction`: has NO_MOVE card
- `ReturnAssistantCardAction(from_tile)`: has RETURN_ASSISTANT card; player has assistant at `from_tile`
- `OneGoodCardAction(good)`: has ONE_GOOD card
- `FiveLiraCardAction`: has FIVE_LIRA card
- `ArrestFamilyCardAction(reward)`: has ARREST_FAMILY card; family not at police station
- `YellowTileAction(from_tile)`: has YELLOW tile; has assistant at `from_tile`; lira >= 2

**Phase 2 actions:**
- `Pay`: other players at current tile; lira >= (players_at_tile - 1) * 2

**Phase 3 tile actions** (all also require being at the right tile):

| Action | Key preconditions |
|--------|------------------|
| `GenericTileAction` at POST_OFFICE | at Post Office (always succeeds) |
| `GenericTileAction` at warehouses | at correct warehouse |
| `GenericTileAction` at WAINWRIGHT | lira >= 7; cart_max < 5; extensions remaining |
| `GenericTileAction` at GEMSTONE_DEALER | `dealer_state.cost is not None`; lira >= cost |
| `GenericTileAction` at FOUNTAIN | at Fountain |
| `FountainAction(locs)` | at Fountain; `locs ⊆ player.assistant_locations`; `locs` non-empty |
| `MosqueAction(good)` | at mosque; player lacks that tile; mosque has that tile available; player has enough goods |
| `BlackMarketAction(good, roll)` | at Black Market; if RedTileAction roll: has red tile; roll sentinel accepted |
| `CaravansaryAction(gains, cost)` | at Caravansary; has `cost` card; discard pile depth >= count of DISCARD gains; not awaiting_discard |
| `MarketAction(goods, new_demand)` | at Small or Large Market; player has those goods; goods ≤ demand for each type; not expecting_demand; new_demand sentinel accepted |
| `TeaHouseAction(call, roll)` | at Tea House; call in {3..12}; if RedTileAction roll: has red tile; roll sentinel accepted |
| `SultansPalaceAction(goods)` | at Sultan's Palace; `required()` not None; goods satisfies required count and per-good minimums |
| `PoliceStationAction(location, sub)` | at Police Station; player has family at police station; location != police_station; `_can_do_tile_action_at(sub, location)` |
| `SkipTileAction` | always valid in phase 3 |
| `GreenTileAction(good)` | has GREEN tile; at a warehouse tile; lira >= 2 |
| `SellAnyCardAction(market_action)` | has SELL_ANY card; at SMALL_MARKET; market_action valid at that tile |
| `DoubleCardAction(card, actions)` | has the card; at the right tile; each sub-action independently valid at that tile |

**Phase 4 actions:**
- `EncounterGovernor(gain, cost, roll)`: governor at current tile; if cost is Pay: lira >= 2; if cost is Card: has that card; roll sentinel accepted
- `EncounterSmuggler(gain, cost, roll)`: smuggler at current tile; if cost is Pay: lira >= 2; if cost is Good: has that good in cart; roll sentinel accepted

### Implementation note

`_can_do_tile_action_at(action, location) -> bool` is a private helper that performs
phase 3 validation at an arbitrary location (used both by the main validator and by the
PoliceStationAction validator for its sub-action). It directly mirrors
`_execute_tile_action_at` in structure.

Because a police station sub-action can itself be a dice-bearing action (e.g.,
`TeaHouseAction(call, UNRESOLVED_ROLL)` sent via the police station), `_can_do_tile_action_at`
must apply the same sentinel-skipping logic as the top-level validator — it must accept
sentinel values in roll and demand parameters without treating them as invalid.

---

## Phase 2: Candidate Generators

`legal_actions()` = filter(`is_valid_action`, `_candidate_actions()`).

`_candidate_actions()` generates a superset of plausible actions for the current
phase, without worrying about whether all of them pass validation. The validator is
the source of correctness; the generator just needs to cover all possibilities.

### Sentinel strategy

Sentinel values are used **only** for parameters that are resolved by external physical
mechanisms (dice rolls, demand card draws from the physical deck). Everything else —
including goods combinations at Sultan's Palace, fountain recall subsets, and market
goods selections — is enumerated in full. This keeps the legal_actions list concrete and
complete, allowing the LLM to make informed strategic choices across all dimensions of
each action.

A module-level `UNRESOLVED_ROLL` and `UNRESOLVED_DEMAND` constant (typed appropriately)
should be defined before implementation. The validator accepts these in roll and demand
parameter positions without performing the specific-value checks that apply to real values.

| Sentinel | Used in | Who resolves it |
|----------|---------|-----------------|
| `UNRESOLVED_ROLL` | `TeaHouseAction.roll`, `BlackMarketAction.roll`, `EncounterGovernor.roll`, `EncounterSmuggler.roll` | Game runner after LLM picks intent |
| `UNRESOLVED_DEMAND` | `MarketAction.new_demand` | Game runner draws demand card after trade |

### By phase

**Phase 1:**
- `Move(loc, skip)` for all 16 locations × {True, False}
- If has EXTRA_MOVE: `ExtraMoveCardAction(Move(loc, skip))` for all 16 × {True, False}
- If has NO_MOVE: `NoMoveCardAction(True)`, `NoMoveCardAction(False)`
- If has RETURN_ASSISTANT: `ReturnAssistantCardAction(loc)` for each location in `assistant_locations`
- Phase-all cards (see below)
- If `yield_required`: only phase-all cards + `YieldTurn()`

**Phase 2:**
- `YieldTurn()`
- `Pay()`
- Phase-all cards

**Phase 3:**
- `SkipTileAction()`
- Tile-specific candidates for current tile (see tile generators below)
- Phase-all cards
- If at a warehouse and has GREEN tile: `GreenTileAction(good)` for each Good (except BLUE)
- If has SELL_ANY and at SMALL_MARKET: `SellAnyCardAction(action)` for each valid market candidate
- If has DOUBLE_SULTAN and at SULTANS_PALACE: `DoubleCardAction(DOUBLE_SULTAN, (a, a))` for valid palace action `a`
- If has DOUBLE_PO and at POST_OFFICE: `DoubleCardAction(DOUBLE_PO, (GenericTileAction(), GenericTileAction()))`
- If has DOUBLE_DEALER and at GEMSTONE_DEALER: `DoubleCardAction(DOUBLE_DEALER, (GenericTileAction(), GenericTileAction()))`

**Phase 4 / outstanding rewards:**
- If `outstanding_reward_choices > 0`: `ChooseReward(LIRA)` + `ChooseReward(card)` for each Card
- If governor at tile: `EncounterGovernor(gain, cost, UNRESOLVED_ROLL)` for each `Card` gain × each (Card in hand + `Pay`) as cost
- If smuggler at tile: `EncounterSmuggler(gain, cost, UNRESOLVED_ROLL)` for each `Good` gain × each (Good with > 0 in cart + `Pay`) as cost
- `YieldTurn()`
- Phase-all cards

**Phase-all cards** (valid in phases 1, 2, 3, 4 when yield_required or not):
- If has ONE_GOOD: `OneGoodCardAction(good)` for each Good
- If has FIVE_LIRA: `FiveLiraCardAction()`
- If has ARREST_FAMILY: `ArrestFamilyCardAction(ChooseReward(choice))` for each choice
- If has YELLOW tile: `YellowTileAction(loc)` for each location in `assistant_locations`

### Tile-specific candidate generators

| Tile | Candidates generated |
|------|---------------------|
| POST_OFFICE | `GenericTileAction()` |
| FABRIC / SPICE / FRUIT WAREHOUSE | `GenericTileAction()` |
| WAINWRIGHT | `GenericTileAction()` |
| FOUNTAIN | `GenericTileAction()` (recall all); `FountainAction(subset)` for each non-empty proper subset of `assistant_locations` (up to 2^5 − 1 = 31 subsets; tractable) |
| GEMSTONE_DEALER | `GenericTileAction()` |
| GREAT_MOSQUE / SMALL_MOSQUE | `MosqueAction(good)` for each Good available at that mosque |
| BLACK_MARKET | `BlackMarketAction(good, UNRESOLVED_ROLL)` for each of {RED, GREEN, YELLOW} |
| CARAVANSARY | `CaravansaryAction(gains, cost)` for each card in hand as cost, each valid gain combination |
| SMALL_MARKET / LARGE_MARKET | `MarketAction(goods, UNRESOLVED_DEMAND)` for each non-empty subset of (player goods ∩ demand) |
| TEA_HOUSE | `TeaHouseAction(call, UNRESOLVED_ROLL)` for call in {3..12} |
| SULTANS_PALACE | `SultansPalaceAction(goods)` for each valid `Counter[Good]` satisfying `required()` given player's cart contents (see note) |
| POLICE_STATION | `PoliceStationAction(loc, sub)` for each non-police-station location × each valid sub-action candidate at that tile |

**Sultan's Palace goods enumeration:** When `required()` contains `None` wildcard slots,
multiple goods combinations are valid (any good can fill a wildcard). With 4 good types,
a cart max of 5 per good, and a small number of wildcards (the count grows slowly as
`required_count` increases), the set of valid combinations is tractable. Enumerate all
`Counter[Good]` that satisfy: total count equals `required_count`; for each specific Good
`g` in `required()`, counter includes at least `required[g]` of `g`; for each Good `g`,
counter does not exceed `cart_contents[g]`.

### Dice and red tile sequencing (open design question)

Several actions encode both a player intent and a random dice outcome in a single
object: `TeaHouseAction(call, roll)`, `BlackMarketAction(good, roll)`. The sentinel
approach handles the common case: the LLM picks the intent, dice are rolled externally,
the runner constructs the final action.

The complication arises when the player has the **red tile**: after seeing the dice
result, the player may modify one die (set to 4) or reroll both. This decision is
strategically dependent on the actual roll, so it cannot be made at the same time as
the intent. The options are:

1. **Two-query protocol:** LLM picks intent (`TeaHouseAction(call=7, UNRESOLVED_ROLL)`),
   runner resolves dice, runner re-queries the LLM with the dice result and asks whether
   to use the red tile. The engine still receives a single atomic action; the splitting
   is at the runner/integration layer only. No changes to action types or JSON schema.

2. **Upfront red tile declaration:** LLM is asked to commit to both the intent and red
   tile usage before dice are rolled (e.g., "I'll call 7 and use red tile to set-to-4 if
   I roll below 4 on either die"). The runner resolves this rule-based. Simpler runner,
   but requires specifying a conditional rather than a direct choice, which may be
   awkward for the LLM.

3. **Intermediate engine actions:** Split `TeaHouseAction` into a call-action and a
   separate optional roll-modification action. Cleaner conceptually but requires changes
   to action types, the JSON schema, and the existing runner/replay machinery.

Option 1 is the path of least resistance and preserves all existing engine behaviour.
Option 3 is the cleanest long-term design but has significant scope. **This choice must
be made before Phase 2 implementation begins** since it affects the action types used
in candidate generation.

---

## Phase 3: Public API and Game State Observation

### `legal_actions() -> list[PlayerAction]`

```python
def legal_actions(self) -> list[PlayerAction]:
    """Return all legal actions available to the current player right now."""
    return [a for a in self._candidate_actions() if self.is_valid_action(a)]
```

### Game state description — `describe()`

Add `GameState.describe() -> dict` producing a structured snapshot of the current state.
The format should be serialisable (JSON-compatible) and complete enough that an LLM can
reason about strategy without access to any other state.

**Fields:**
```
current_player:
  color, phase, lira, rubies
  cart: {red, blue, green, yellow, cart_max}
  hand: {card_name: count, ...}
  assistant_locations: [location_numbers]
  family_location: location_number

other_players: [{color, location, rubies}, ...]   # public info only

board:
  governor_location: location_number
  smuggler_location: location_number
  tile_layout: {location_number: tile_name, ...}

tiles:
  small_market: {demand: {good: count}, one_cost: int}
  large_market: {demand: {good: count}, one_cost: int}
  post_office: {position: int, next_goods: [good_names], next_lira: int}
  great_mosque: {available_tiles: {good: cost}, ...}
  small_mosque: {available_tiles: {good: cost}, ...}
  sultans_palace: {required_count: int, required: {good: count}}
  gemstone_dealer: {cost: int | null}
  wainwright: {extensions_remaining: int}
  caravansary: {discard_pile_top: card_name | null}

outstanding_reward_choices: int
```

### LLM prompt guide

In addition to `describe()`, the integration layer should provide a static markdown
explanation of the action format and sentinel values as part of the system prompt or
alongside the legal actions list. At minimum it should explain:

- What `UNRESOLVED_ROLL` means and that the runner will prompt for the dice result
- What `UNRESOLVED_DEMAND` means and that the new demand will be input after the trade
- How to interpret `FountainAction(locations)` vs `GenericTileAction()` at the Fountain
- The two-step resolution protocol for red tile decisions (once that is resolved above)

The legal actions list itself may also be offered as full enumeration on request — if
the LLM's response does not match a legal action, the runner can re-query with the
explicit list rather than failing immediately.

---

## Testing

All phases must be covered by tests:

- **Phase 0:** Existing tests should pass unchanged after refactor.
- **Phase 1:** Unit tests for `is_valid_action` covering valid and invalid cases for each
  action type. Verify that `is_valid_action` agrees with `take_action` (no action that
  passes the validator should raise when taken, and vice versa) — a property test is
  well-suited here.
- **Phase 2:** `legal_actions()` is tested against known game states, verifying no legal
  action is omitted and no illegal action is included.
- **Phase 3:** `describe()` is a snapshot test.

---

## File layout

New code belongs in the `istanbul_game` package:

| File | Contents |
|------|----------|
| `istanbul_game/game.py` | `is_valid_action`, `_can_do_tile_action_at`, `_execute_tile_action_at`, `_candidate_actions`, `legal_actions`, `describe`; refactored handlers |
| `istanbul_game/candidates.py` | Candidate generator helpers (if game.py becomes unwieldy) |
| `tests/test_validator.py` | Validator unit tests |
| `tests/test_legal_actions.py` | Legal actions integration tests |

The phase 0 police station refactor stays entirely within `game.py`.
