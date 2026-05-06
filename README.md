# ChronicleJSON Format

The **chronicle** is the self-contained JSON representation of a finished (or in-progress) [Blood on the Clocktower](https://bloodontheclocktower.com) game. It bundles the script name, the roster, the winner, and a stream of phases, each with its own events and Storyteller notes. Enough to reconstruct the game or narrate it elsewhere.

## Status

* Version: 0.5
* Used in production by [Blocktower](https://apps.apple.com/us/app/blocktower-botc-toolkit/id6758778915) and [Slaydate](https://slaydate.app).
* Expect breaking changes before 1.0.

## Top-level shape

```json
{
  "script": "Trouble Brewing",
  "storytellers": [
    { "name": "Steve", "id": "abc-123" }
  ],
  "winner": "good",
  "players": [ … ],
  "phases": [ … ]
}
```

| Field | Type | Notes |
|-------|------|-------|
| `script` | string \| `ScriptMetadata` \| null | The script played. A bare string is shorthand for `{ "name": "..." }` with all other fields null; use the object form to carry author, version, external id, or url. See below. Null for games without a known script. |
| `storytellers` | array of `Storyteller` | The humans who ran the game. Order is informational, not significant. Empty array when no Storyteller info is available. See below. |
| `winner` | `"good"` \| `"evil"` \| null | The Storyteller's recorded winner. Null for games ended without declaring a winner. |
| `players` | array of `PlayerRecord` | Final roster, in position order (which equals initial seating). See below. |
| `phases` | array of `ChroniclePhase` | Events grouped by phase, in play order. See below. |

## `ScriptMetadata`

```json
{
  "name": "Trouble Brewing",
  "author": "The Pandemonium Institute",
  "version": "1.4.0",
  "id": "tb",
  "url": "https://botcscripts.com/script/tb"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `name` | string \| null | Human-readable script name. Null when no canonical name is available (homebrew, work-in-progress). |
| `author` | string \| null | Script author. Null when unknown or omitted. |
| `version` | string \| null | Free-form version string (e.g. `"1.4.0"`, `"draft-2"`). Opaque to ChronicleJSON. |
| `id` | string \| null | Optional external identifier for the script (e.g. a botcscripts id). Opaque to ChronicleJSON; not required to be a UUID. Omit or set to null when no such id is available. |
| `url` | string \| null | Optional URL pointing to a published version of the script. |

The top-level `script` field accepts a bare string as shorthand for a `ScriptMetadata` object with only `name` set. Use the object form whenever you have more than just a name.

## `Storyteller`

```json
{
  "name": "Steve",
  "id": "abc-123"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `name` | string \| null | Storyteller's name. Null when unnamed. |
| `id` | string \| null | Optional external identifier for the Storyteller, carried through if the producing system has one (account id, roster id, etc). Opaque to ChronicleJSON; not required to be a UUID. Omit or set to null when no such id is available. |

## `PlayerRecord`

```json
{
  "position": 3,
  "name": "Carol",
  "characters": ["imp"],
  "alignment": "evil"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `position` | integer | Stable per-player identifier, **1-based**. Assigned when the player is added (at game start or via `playerAdded`) and never changes for that player, including after `playerMoved` events. Distinct from a player's current *seat* (where they're sitting in the circle). `players[0]` has `position: 1`; the array is in position order. |
| `id` | string \| null | Optional external identifier for the player, carried through if the producing system has one (account id, roster id, etc). Opaque to ChronicleJSON; not required to be a UUID. Omit or set to null when no such id is available. Events still reference players by `position`, not `id`. |
| `name` | string \| null | Player's name. Null for unnamed seats. |
| `characters` | array of string | Every character the player held, in time order, as character ids (see below). The final entry is the player's current / final character; an empty array means they were never assigned one. |
| `alignment` | `"good"` \| `"evil"` \| null | The player's current / final alignment. Null when they were never assigned an alignment. Alignment usually tracks the character's team, but may diverge (a Goon, Pit Hag shenanigans, etc). Reconstruct the full alignment history from `alignmentChange` events. |

### Character ids

Characters are identified by their lowercase, no-separator id, not their display name: `"scarletwoman"`, not `"Scarlet Woman"`; `"pithag"`, not `"Pit Hag"`. This matches the convention used by official resources (e.g. https://release.botc.app/resources/).

## `ChroniclePhase`

```json
{
  "phase": "day",
  "number": 1,
  "notes": "Poisoner claimed Chef.",
  "events": [ … ]
}
```

| Field | Type | Notes |
|-------|------|-------|
| `phase` | `"day"` \| `"night"` | Which half of the day/night cycle this section represents. |
| `number` | integer | In-game phase number. Day 1 is the first day; night `N` follows day `N - 1` (so `night: 1` precedes `day: 1` only in traveller setups; typical play starts at `day: 1`). |
| `notes` | string \| null | Storyteller's free-text notes for this phase. `null` when the Storyteller left the phase unnoted. |
| `events` | array of `GameLogEntry` | Events that happened inside this phase, in insertion order. |

Phases appear in the order they occurred. A phase that carries no events and has `null` notes is elided from the output. Retreating to a previous phase and then acting writes into that phase's existing `events` array; it doesn't produce a duplicate phase block.

## `GameLogEntry`

```json
{
  "id": "B6B9E5B2-…",
  "timestamp": "2026-04-23T10:00:00.123Z",
  "nominationForExecution": {
    "nominatorPosition": 4,
    "nomineePosition": 3
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | string (UUID) | Per-entry identifier. Stable across decode/encode, and preserved when a death event is reclassified. |
| `timestamp` | string (ISO-8601) | UTC timestamp with millisecond precision (`YYYY-MM-DDTHH:mm:ss.SSSZ`). A hand-authored payload may omit the fractional seconds. |
| *(event key)* | object | Exactly one of the variant keys listed below. The key names the variant; the value is an object whose fields are that variant's labels. |

Entries are written in insertion order within each phase. A recorded death may be reclassified after the fact, which rewrites the entry's event value in place (preserving `id` and `timestamp`). Treat the log as "mostly append-only, with late corrections to death events."

## Event variants

Each entry carries exactly one event-variant key. Unknown variants should be skipped defensively, the set may grow over time and older parsers won't recognize newer events.

Each variant below shows only the event key and its value, as it would appear nested inside a `GameLogEntry`. The surrounding `id` and `timestamp` fields are omitted for brevity.

### `gameStart`

```json
"gameStart": {
  "playerCount": 5
}
```

| Field | Type | Notes |
|-------|------|-------|
| `playerCount` | integer | Seats at game start, including travellers. |

### `gameEnd`

```json
"gameEnd": {
  "winner": "good"
}
```

`winner` is `"good"`, `"evil"`, or `null`.

### `nominationForExecution`

```json
"nominationForExecution": {
  "nominatorPosition": 4,
  "nomineePosition": 3
}
```

Standard day-phase nomination targeting an in-play player for execution. Positions are 1-based; names are resolved via `players[]`. The companion `voteResult` carries an `outcome` of `"onTheBlock"`, `"failed"`, `"tied"`, or `"executed"`.

### `nominationForExile`

```json
"nominationForExile": {
  "nominatorPosition": 4,
  "nomineePosition": 6
}
```

Traveller-exile nomination. Structure mirrors `nominationForExecution`, the two variants exist as separate events so consumers don't need to branch on a flag. The companion `voteResult` carries an `outcome` of `"exiled"` or `"failed"`.

### `voteResult`

```json
"voteResult": {
  "nomineePosition": 3,
  "votes": 3,
  "required": 2,
  "outcome": "onTheBlock"
}
```

`outcome` is a `NominationOutcome`:

| Value | Meaning |
|-------|---------|
| `"pending"` | Vote hasn't resolved yet. Rare in the log, typically set immediately before being overwritten. |
| `"failed"` | Didn't meet the threshold and didn't tie. |
| `"tied"` | Matched the current block-leader's vote count, the block was cleared, nobody gained the block. |
| `"onTheBlock"` | Became the new block leader. |
| `"executed"` | Resolved to execution at end of day (vote path). Whether the player actually died is recorded by a separate `death` event. Abilities like Devil's Advocate can prevent the death without changing this outcome. |
| `"exiled"` | Resolved to exile (traveller path). Whether the traveller actually died is recorded by a separate `death` event. Abilities like a funny Deviant can prevent the death without changing this outcome. |

`required` reflects the threshold at the moment of resolution and can rise mid-day to beat the current block leader.

### `death`

```json
"death": {
  "playerPosition": 3,
  "reason": "executed"
}
```

Every death the Storyteller records produces exactly one `death` entry. `reason` is one of:

| Value | Meaning |
|-------|---------|
| `"executed"` | Vote-path execution at end of day: the on-the-block player was confirmed dead. |
| `"exiled"` | Vote-path exile: the traveller was confirmed dead from exile. |
| `"killed"` | Storyteller-directed death during a night phase (typically a demon kill). |
| `"died"` | Any other death: Ravenkeeper trigger, Virgin ability, reclassified execution, etc. |

The Storyteller may reclassify a death after the fact, this rewrites the `reason` field in place and preserves the entry's `id` and `timestamp`.

### `resurrection`

```json
"resurrection": {
  "playerPosition": 2
}
```

Storyteller brought a dead player back to life.

### `deadVoteUsed`

```json
"deadVoteUsed": {
  "playerPosition": 2
}
```

A dead player has spent their one-shot dead vote token. Usually emitted alongside a `nominationForExecution` the player voted on, but the Storyteller may record it at any time, including outside a vote, e.g. as a penalty or to reflect a side-effect of an ability. The event stands alone and is not tied to a specific `voteResult`. Mis-recorded entries are corrected by removing the entry, not by emitting a counter-event.

### `characterChange`

```json
"characterChange": {
  "playerPosition": 2,
  "from": "washerwoman",
  "to": "drunk"
}
```

Use for mid-game swaps (script abilities, Pit Hag, Storyteller corrections, etc). `from` and `to` are character ids. `to` may be null if the Storyteller clears a character; `from` is the prior character (nullable in the schema, but in practice always present).

### `alignmentChange`

```json
"alignmentChange": {
  "playerPosition": 2,
  "from": "good",
  "to": "evil"
}
```

The player's alignment changes independently of their character, e.g. Goon, Pit Hag shenanigans or the Storyteller simply correcting a misrecorded alignment. Initial alignments are captured on `players[]` and do not produce events. Both `from` and `to` are `"good"`, `"evil"`, or `null`.

### `playerAdded`

```json
"playerAdded": {
  "position": 12,
  "seat": 4
}
```

A new player joined mid-game (typically a traveller). `position` is the stable per-player identifier assigned to the new player, 1-based and never reused. `seat` is the seat the new player was inserted into, 1-based; existing players at that seat or higher shift up by one to make room. Subsequent `playerMoved` events may relocate the new player.

### `playerRemoved`

```json
"playerRemoved": {
  "position": 6
}
```

A player left active play (typically an exiled traveller leaving the game, but also covers an IRL drop-out or a Storyteller correction). `position` identifies the leaving player, 1-based. Positions are not reused after removal. Existing players at higher seat numbers shift down by one to close the gap, keeping seats contiguous.

Independent of `death`: a removed player may or may not have a corresponding death entry, and a dead player may remain seated. For example, an exiled traveller who keeps their seat (and dead vote) produces a `death` entry but no `playerRemoved`; a a funny Deviant who survives the exile vote but leaves the game produces a `playerRemoved` but no `death`.

### `playerMoved`

```json
"playerMoved": {
  "moves": [
    { "playerPosition": 3, "toSeat": 5 },
    { "playerPosition": 5, "toSeat": 7 },
    { "playerPosition": 7, "toSeat": 3 }
  ]
}
```

One or more players changed seats simultaneously (typically a Matron swap). The whole permutation is recorded as a single event so consumers don't observe an invalid intermediate state. `playerPosition` references the player by their stable position; `toSeat` is the destination seat, 1-based. After the event resolves, each seat is occupied by at most one player.

`playerMoved` is the only event that mutates the seat of an existing player. `position` is unchanged for every player involved.

## Decoding tips

- **Unknown event variants**: treat the set of event variants as closed-but-extensible. Decode defensively and skip unrecognized single-key objects.
- **Exiles**: a successful exile vote produces a `voteResult` with `outcome: "exiled"`. If the traveller dies, a `death` entry with `reason: "exiled"` follows. If the traveller leaves the game, a `playerRemoved` entry follows. These are independent. Abilities like a funny Deviant can break the usual "exiled, dead, removed" chain.
- **Positions vs. seats**: `position` is the stable per-player identifier, assigned at add time, never changes, used by every event that references a player. *Seat*, where the player is sitting in the circle, is initialized when the player is added (game-start players sit at seat == position; mid-game adds use `playerAdded.seat` and shift other seats), and changes via `playerMoved`. The `players[]` array is ordered by `position`, not by current seat. To render the current circle, walk `playerAdded`, `playerRemoved`, and `playerMoved` events and apply the resulting seat assignments.
- **Positions**: 1-based throughout. Positions may be non-contiguous after `playerRemoved` events and are never reused.
- **Timestamps**: ISO-8601 UTC. Parse leniently, milliseconds may be absent on hand-authored payloads.
- **Grouping**: events are already grouped by phase. Iterate `phases[*].events` in order to walk the game chronologically. The phase's `phase` + `number` disambiguate same-numbered sections.

## Complete example

A short 5-player Trouble Brewing game: nobody dies Day 1, Imp kills the Empath Night 1, the Imp is nominated and executed Day 2.

```json
{
  "script": "Trouble Brewing",
  "storytellers": [{ "name": "Steve", "id": null }],
  "winner": "good",
  "phases": [
    {
      "events": [
        {
          "gameStart": { "playerCount": 5 },
          "id": "6C6E0F80-0000-4000-8000-000000000001",
          "timestamp": "2026-04-23T10:00:00.000Z"
        }
      ],
      "notes": null,
      "number": 1,
      "phase": "day"
    },
    {
      "events": [
        {
          "death": { "playerPosition": 2, "reason": "killed" },
          "id": "6C6E0F80-0000-4000-8000-000000000003",
          "timestamp": "2026-04-23T10:17:00.000Z"
        }
      ],
      "notes": "Imp chose Empath. Poisoner skipped the Imp; no bluff to protect.",
      "number": 2,
      "phase": "night"
    },
    {
      "events": [
        {
          "nominationForExecution": {
            "nominatorPosition": 4,
            "nomineePosition": 3
          },
          "id": "6C6E0F80-0000-4000-8000-000000000004",
          "timestamp": "2026-04-23T10:30:00.000Z"
        },
        {
          "voteResult": {
            "nomineePosition": 3,
            "outcome": "onTheBlock",
            "required": 2, "votes": 3
          },
          "id": "6C6E0F80-0000-4000-8000-000000000005",
          "timestamp": "2026-04-23T10:31:00.000Z"
        },
        {
          "death": { "playerPosition": 3, "reason": "executed" },
          "id": "6C6E0F80-0000-4000-8000-000000000006",
          "timestamp": "2026-04-23T10:35:00.000Z"
        },
        {
          "gameEnd": { "winner": "good" },
          "id": "6C6E0F80-0000-4000-8000-000000000008",
          "timestamp": "2026-04-23T10:35:15.000Z"
        }
      ],
      "notes": "Poisoner claimed Chef, Empath's death spoke for itself, nomination landed hard.",
      "number": 2,
      "phase": "day"
    }
  ],
  "players": [
    { "alignment": "good", "characters": ["washerwoman"], "name": "Alice", "position": 1 },
    { "alignment": "good", "characters": ["empath"],      "name": "Bob",   "position": 2 },
    { "alignment": "evil", "characters": ["imp"],         "name": "Carol", "position": 3 },
    { "alignment": "evil", "characters": ["poisoner"],    "name": "Dan",   "position": 4 },
    { "alignment": "good", "characters": ["virgin"],      "name": "Eve",   "position": 5 }
  ]
}
```

# Contributing

Found a gap, an ambiguity, or a variant that should exist? Open an issue. Pull requests welcome for typo fixes and clarifications; for semantic changes, let's discuss in an issue first.

# License

Licenced under MIT. See LICENSE.

ChronicleJSON is an unofficial community format. Blood on the Clocktower is a trademark of The Pandemonium Institute. This project is not affiliated with, endorsed by, or sponsored by The Pandemonium Institute.
