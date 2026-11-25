# Poker: Networked Five-Card Draw

## 1. Project overview

The server owns the whole game: it shuffles the deck, deals cards, validates every move, decides the winner and settles the pot. Clients only send commands and render the events they receive. A client can never declare its own cards or learn another player's hand before the rules allow it.

## 2. Modules

| Module | Responsibility | Depends on |
|---|---|---|
| `poker-model` | Cards, deck, players, game state and engine, domain validation, hand evaluation, payout | – |
| `poker-common` | Types needed by both sides: protocol messages, parser, serializer | `poker-model` (only if really needed) |
| `poker-server` | Connections, sessions, player ↔ connection binding, command handling, event broadcast | `poker-model`, `poker-common` |
| `poker-client` | Text UI, sending commands, receiving server events | `poker-common` |

## 3. Game rules (contract)

Variant: **five-card draw**, 2–4 players, fixed-limit betting.
Sources: [Poker pięciokartowy dobierany (Wikipedia)](https://pl.wikipedia.org/wiki/Poker_pi%C4%99ciokartowy_dobierany).

### 3.1 Deck and cards
- Standard 52-card deck, no jokers. Ranks 2–A, four suits.
- `Rank`, `Suit` and `Card` are immutable value objects. Suit has **no** effect on hand strength.
- The deck is shuffled by the server only. In tests a fixed deck order or a seeded `Random` is injected.
- No card appears twice in the deck or across hands, the stock and the discard pile.

### 3.2 Hand ranking (high to low)
1. Straight flush (a royal flush is simply the highest straight flush)
2. Four of a kind
3. Full house
4. Flush
5. Straight
6. Three of a kind
7. Two pair
8. One pair
9. High card

**Ace:** high (A-K-Q-J-10) or low (A-2-3-4-5, the "wheel", the lowest straight). No wrap-around (Q-K-A-2-3 is **not** a straight).

**Comparison:** first by category, then by the tie-breaking values in that category, in order of importance (e.g. for one pair: pair rank, then the three kickers in descending order). Suits are never used to break ties. For the wheel the straight's top card is the 5.

**Tie:** hands with the same category and the same tie-breaking values are equal.

### 3.3 Chips, ante and betting
| Parameter | Default | Notes |
|---|---|---|
| Starting chips per player | 100 | Reset at the start of every game |
| Ante | 1 | Paid by every player into the pot before the deal |
| Bet size | 5 | Fixed-limit: every bet or raise is exactly this amount |
| Max raises per betting round | 2 | After one bet and two raises the round is capped |

- Two betting rounds: **BET1** (after the deal) and **BET2** (after the draw).
- The player in seat 0 acts first; order proceeds by seat. A fold removes the player from the rest of the hand.
- A betting round closes when every non-folded player has acted and all of them have put in equal amounts.
- **Insufficient chips [simplification]:** there is no all-in and there are no side pots. A player who cannot afford a call, bet or raise may only fold. The configuration is validated so this cannot happen with legal play:
  `startingChips ≥ ante + 2 × (1 + maxRaises) × betSize` (with defaults: `100 ≥ 1 + 2×3×5 = 31`). A `CREATE` that violates it is rejected.

Simplification was allowed by the teacher on the Microsoft Teams chat.

### 3.4 Draw
- Each active player, in seat order, discards **0–3** cards (0 = stand pat) and receives the same number from the stock.
- Discarded cards go to the discard pile and **do not return** to the stock during the hand.
- A draw with an out-of-range index, a duplicate index, or more than 3 cards is rejected **entirely**; the hand and the deck stay unchanged (no partial exchange).

### 3.5 End of a hand and settlement
- **Everyone else folds:** the last remaining player wins the pot immediately. No draw and no showdown happens.
- **Showdown:** after BET2 all non-folded players reveal their hands; the best hand wins the whole pot.
- **Tie:** the pot is split equally between the tied players. Indivisible remainder chips go one each to the tied players in seat order starting from seat 0. No chip is created or lost.
- **Chip invariant (checked after every payout):** `sum(player stacks) + pot == players × startingChips`.

### 3.6 Game flow

```
LOBBY -> ANTE -> DEAL -> BET1 -> DRAW -> BET2 -> SHOWDOWN -> PAYOUT -> END

Shortcut: in BET1 / DRAW / BET2, when only one player remains -> PAYOUT -> END
Disconnect: a player who disconnects is treated as a fold (see 3.8)
```

### 3.7 Action table (engine level)

| Action | Phase | Who | Conditions | Effect |
|---|---|---|---|---|
| JOIN | LOBBY | any new player | free seat, game not full | Seat assigned, player ID issued |
| CHECK | BET1 / BET2 | player to move | no pending bet | Turn passes |
| BET | BET1 / BET2 | player to move | no pending bet, enough chips | Pot += bet size, bet becomes pending |
| CALL | BET1 / BET2 | player to move | bet pending, enough chips | Pot += difference |
| RAISE | BET1 / BET2 | player to move | bet pending, raise cap not reached, enough chips | Pot += call + raise |
| FOLD | BET1 / BET2 | player to move | – | Player leaves the hand |
| DRAW | DRAW | player to move | ≤ 3 distinct valid indices | Cards replaced, turn passes |

Every action is validated **before** any state change. A rejected action leaves the hand, the pot and the turn exactly as they were.

### 3.8 Disconnects
- During **LOBBY:** the player's seat is freed.
- During a **hand:** the player is treated as having folded; others are notified; if one player remains, they win the pot.
- The server never hangs, closes the player's resources, and stays ready for the next game.
- Automatic reconnection, heartbeat and request throttling are out of scope.

### 3.9 Visibility
| Information | Visible to |
|---|---|
| Own hand | Owner only |
| Number of cards drawn by others | Everyone |
| Pot, stacks, phase, whose turn | Everyone |
| Hands at showdown | Everyone, only for players who did not fold |
| Deck order, stock, discard pile | Nobody (server only) |

## 4. Protocol (summary)

Text, UTF-8, one message per line (`\n`). A message may arrive split across reads, or several may arrive in one read; the parser must not assume one read equals one message.

Messages before IDs are assigned: `CREATE`, `JOIN`, and their confirmations. After that: `GAME_ID PLAYER_ID ACTION [PARAMS...]`.
The player ID sent by a client is checked against the connection it came from; it does not authorize a move for another player.

The full table (direction, parameters and types, allowed phase, success reply, errors) will be in `docs/protocol.md` (TODO).
Syntax errors are reported by the parser; well-formed but illegal moves (wrong turn, wrong phase, invalid stake or draw) come back from the engine as separate domain errors.

## 5. Building, running and testing

### Requirements
JDK 21 or 25, Maven 3.9+.

### Build (outside the IDE)
```
mvn clean verify
```
This runs unit and integration tests, generates the JaCoCo coverage report and builds the executable JARs.

### Executable JARs
```
poker-server/target/poker-server-<version>-jar-with-dependencies.jar
poker-client/target/poker-client-<version>-jar-with-dependencies.jar
```
Exact file names depend on the POM version; to be confirmed after the first successful build.

### Run
```
# server: port and default number of players per game (2-4)
java -jar poker-server/target/poker-server-<version>-jar-with-dependencies.jar --port 5000 --players 3

# client (one per player)
java -jar poker-client/target/poker-client-<version>-jar-with-dependencies.jar --host localhost --port 5000
```
- Default players per game: **3** (server argument `--players`, allowed 2–4). To be adjusted once implemented.
- Number of players for the next game can be changed with `CREATE <players>`, so a 3-player game and then a 4-player game run without restarting the server. The number is fixed once a game has started.

### Example session

TODO

### Documentation and reports
```
mvn javadoc:javadoc          # generated API docs
```

## 6. Tests

| ID | Scope | Status |
|---|---|---|
| P-T1 | Deck and deal: 52 unique cards; 2, 3 and 4 players without duplicates; draw per the rules | ☐ |
| P-T2 | Hand evaluation: all categories, in-category comparison, kickers, ties, ace high/low, invalid straights | ☐ |
| P-T3 | Full hand: phases, ante, betting, end by folds or showdown, chip conservation | ☐ |
| P-T4 | Domain errors: out of turn/phase, bad stake, no chips, invalid or duplicate draw indices; state unchanged | ☐ |
| P-T5 | Parser and replies: every message type, missing/invalid parameters, unknown action or ID | ☐ |
| P-T6 | Permissions and privacy: impersonation rejected; others' cards hidden | ☐ |
| P-T7 | TCP integration: real server, at least 2 test clients | ☐ |
| P-T8 | Consecutive games (3 then 4 players) in one server process, no leftover state | ☐ |
| P-T9 | Disconnect and cleanup: no hang, sockets and tasks closed | ☐ |
| P-T10 | Extensions: NIO split/batched messages; multi-game isolation | ☐ |

Test rules: no sockets or manual input in model tests (P-T1–P-T4); fixed deck order or seeded `Random` instead of "lucky" shuffles; network tests have timeouts and release sockets and threads even when an assertion fails.

**Coverage:** minimum 70% of lines of the assessed logic (model/rules, parser and protocol handling, validation, game control, and implemented extensions).

**Coverage exclusions:**
- `Main` classes of server and client (startup only)
- Pure output formatting
- Generated or logic-free code

## 7. Extensions

- **P6, java.nio:** planned
- **P7, multiple simultaneous games:** planned

## 8. Assumptions and limitations

- One game = one hand; chips reset for each new game.
- No all-in or side pots (see 3.3).
- No GUI, no database, no reconnection to an interrupted session.
- Test data (fixed decks) are available only in tests; they are not a way for a client to choose its cards or see the deck.
