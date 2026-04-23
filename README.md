# ChronicleJSON Format

The **chronicle** is the self-contained JSON representation of a finished (or in-progress) [Blood on the Clocktower](https://bloodontheclocktower.com) game. It bundles the script name, the roster, the winner, and a stream of phases, each with its own events and Storyteller notes. Enough to reconstruct the game or narrate it elsewhere.

## Status

* Version: 0.1
* Used in production by [Blocktower](https://apps.apple.com/us/app/blocktower-botc-toolkit/id6758778915) and [Slaydate](https://slaydate.app).
* Expect breaking changes before 1.0.

## Top-level shape

```json
{
  "scriptName": "Trouble Brewing",
  "winner": "good",
  "players": [ … ],
  "phases": [ … ]
}
```

| Field | Type | Notes |
|-------|------|-------|
| `scriptName` | string \| null | Human-readable script name. Null for unnamed/custom scripts. |
| `winner` | `"good"` \| `"evil"` \| null | The Storyteller's recorded winner. Null for games ended without declaring a winner. |
| `players` | array of `PlayerRecord` | Final roster. See below. |
| `phases` | array of `ChroniclePhase` | Events grouped by phase, in play order. See below. |

## `PlayerRecord`

```json
{
  "position": 3,
  "name": "Carol",
  "characters": ["Imp"],
  "alignment": "evil"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `position` | integer | Seat index in the circular layout, **1-based**. `players[0]` has `position: 1`. |
| `name` | string \| null | Player's name. Null for unnamed seats. |
| `characters` | array of string | Every character the player held, in time order. The final entry is the player's current / final character; an empty array means they were never assigned one. |
| `alignment` | `"good"` \| `"evil"` \| null | The player's current / final alignment. Null when they were never assigned an alignment. Alignment usually tracks the character's team, but may diverge (a Goon, Pit Hag shenanigans, etc). Reconstruct the full alignment history from `alignmentChange` events. |

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
| `"executed"` | Was executed at end of day (vote path). |
| `"exiled"` | Was exiled (traveller path). |

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

### `characterChange`

```json
"characterChange": {
  "playerPosition": 2,
  "from": "Washerwoman",
  "to": "Drunk"
}
```

Use for mid-game swaps (script abilities, Pit Hag, Storyteller corrections, etc). `to` may be null if the Storyteller clears a character; `from` is the prior character (nullable in the schema, but in practice always present).

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
  "position": 6
}
```

A seat was inserted mid-game (typically a traveller). `position` is 1-based.

## Decoding tips

- **Unknown event variants**: treat the set of event variants as closed-but-extensible. Decode defensively and skip unrecognized single-key objects.
- **Exile deaths**: a successful exile nomination produces a `voteResult` with `outcome: "exiled"`. The traveller should be transitioned. No `death` entry is written for the exile itself.
- **Positions**: 1-based throughout (nominations, player records, `playerAdded`). Seats may be non-contiguous if players were added mid-game or relocate for other reasons, but the array order in `players` still matches the circular layout.
- **Timestamps**: ISO-8601 UTC. Parse leniently, milliseconds may be absent on hand-authored payloads.
- **Grouping**: events are already grouped by phase. Iterate `phases[*].events` in order to walk the game chronologically. The phase's `phase` + `number` disambiguate same-numbered sections.

## Complete example

A short 5-player Trouble Brewing game: nobody dies Day 1, Imp kills the Empath Night 1, the Imp is nominated and executed Day 2.

```json
{
  "scriptName": "Trouble Brewing",
  "winner": "good",
  "phases": [
    {
      "events": [
        {
          "gameStart": { "playerCount": 5 },
          "id": "6C6E0F80-0000-4000-8000-000000000001",
          "timestamp": "2026-04-23T10:00:00.000Z"
        },
        {
          "grimoirePhoto": { "timing": "start" },
          "id": "6C6E0F80-0000-4000-8000-000000000002",
          "timestamp": "2026-04-23T10:00:10.000Z"
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
          "grimoirePhoto": { "timing": "end" },
          "id": "6C6E0F80-0000-4000-8000-000000000007",
          "timestamp": "2026-04-23T10:35:10.000Z"
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
    { "alignment": "good", "characters": ["Washerwoman"], "name": "Alice", "position": 1 },
    { "alignment": "good", "characters": ["Empath"],      "name": "Bob",   "position": 2 },
    { "alignment": "evil", "characters": ["Imp"],         "name": "Carol", "position": 3 },
    { "alignment": "evil", "characters": ["Poisoner"],    "name": "Dan",   "position": 4 },
    { "alignment": "good", "characters": ["Virgin"],      "name": "Eve",   "position": 5 }
  ]
}
```

# Contributing

Found a gap, an ambiguity, or a variant that should exist? Open an issue. Pull requests welcome for typo fixes and clarifications; for semantic changes, let's discuss in an issue first.

# License

Licenced under MIT. See LICENSE.
