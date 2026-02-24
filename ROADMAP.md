# Roadmap

## Goal

Enable a full game session where each player is independently controlled by either a
human at a terminal or an LLM. The physical game (board, tiles, pieces, dice, card decks)
remains the authoritative source for random elements; the engine records and validates
the game sequence and drives LLM turns.

---

## Milestones

### 1. LLM-ready engine interfaces *(in progress — see `designs/DESIGN-LLM-PLAYER.md`)*

Before any interactive play is possible, the engine needs to be able to report what
actions are legal at any moment and describe the current game state in a way an LLM
can reason about.

- Refactor police station handler to remove temporary player relocation
- `GameState.is_valid_action(action) -> bool` — pure predicate, no crash on invalid input
- `GameState.legal_actions() -> list[PlayerAction]` — exact set of legal actions now
- `GameState.describe()` — structured game state snapshot for LLM consumption
- Sentinel value strategy for externally-resolved parameters (dice rolls, demand draws,
  fountain recall, sultan's palace wildcard goods)

### 2. Interactive game runner

The existing runner replays a pre-recorded CSV; we need one that drives play forward
interactively, prompting each player for their move in turn.

- Terminal-based runner that loops over turns and phases
- Game state display after each action (human-readable, not just JSON)
- Human player input: present legal actions as a numbered menu or accept structured text
- Configuration: specify which player colors are human vs. LLM at startup

### 3. Physical input handling

Many game events are resolved by physical components (dice, card decks). The interactive
runner must prompt the operator to provide these values and feed them back into the engine
as the concrete action parameters that currently use sentinel values.

- After an action with an `UNRESOLVED_ROLL` sentinel is chosen: prompt for dice result
- After a market sale with `UNRESOLVED_DEMAND` sentinel: prompt for the new demand card drawn
- Caravansary card draws: prompt for which card(s) are available from the physical deck top
- Game setup: prompt for initial tile layout, governor/smuggler starting positions, initial
  player cards

**Open design question — card deck model:**
The game uses a shared card deck (Caravansary draws, initial player hands) and a separate
market demand deck. Two main options:

- *Fully physical*: the deck is never modelled digitally; every draw prompts the operator
  to draw and input. Simple, no shuffling logic needed, works naturally with the physical game.
- *Digital tracking*: the engine tracks deck contents and validates/randomises draws. Needed
  if playing without a physical copy (e.g., pure LLM vs. LLM games).

For the near-term goal of augmenting a physical game session, the fully-physical approach
is likely the path of least resistance. The deck model design should be revisited before
milestone 3 implementation begins.

### 4. Takeback support

Allowing a move to be undone is important for human usability and for correcting LLM
mistakes or mis-inputs.

The path of least resistance: the runner records the full ordered sequence of actions
taken since game start (externally-resolved values baked in). A takeback replays the
sequence from the initial state, omitting the last N actions. This is O(game length)
but games are short enough that this is acceptable; checkpoint-based approaches can be
considered later if needed.

- Action history log maintained by the runner
- `takeback` command (or menu option) available at any input prompt
- Specify how many actions to undo (default 1); display the action(s) being removed before
  confirming
- After replay, resume interactive play from the restored state

### 5. LLM player integration

Wire up the LLM API to drive player turns automatically when a player's color is
configured as LLM-controlled.

- Call `legal_actions()` and `describe()` to construct the LLM prompt
- Parse the LLM's response to identify which action was chosen
- Prompt the operator to resolve any sentinel values in the chosen action (dice roll,
  card draw) before submitting to `take_action()`
- Retry or surface an error if the LLM selects an action that fails validation (should be
  rare once the candidate generator is correct, but the interface must not crash)
- Configuration: model selection, API key handling

---

## Cross-cutting notes

- **Replay compatibility:** action history entries must be serialisable in the existing
  JSON/CSV formats so that a session can be saved and resumed or analysed post-game.
- **Mixed sessions:** a single game can have any combination of human and LLM players;
  the runner treats them identically except for how the move is obtained.
- **Dice and cards for LLM players:** when an LLM player selects a dice-bearing action,
  the operator rolls the physical dice (or the engine prompts for input) just as for a
  human player — the LLM selects the intent, the physical world provides the outcome.
