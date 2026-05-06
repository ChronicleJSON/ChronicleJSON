# Changelog

## 0.5

- **Breaking:** Characters are now identified by their lowercase, no-separator id (e.g. `"scarletwoman"`) rather than their display name (`"Scarlet Woman"`). Affects `PlayerRecord.characters` and `characterChange.from` / `to`. This matches official resources (https://release.botc.app/resources/) and lets frontends localize display names without changing the data format.
- **Breaking:** Renamed top-level `scriptName` to `script`, which now accepts either a string (just the name, as before) or a `ScriptMetadata` object with optional `author`, `version`, external `id`, and `url`.
- **Breaking:** Exile is now recorded with the same death/removal granularity as execution. A successful exile vote still produces a `voteResult` with `outcome: "exiled"`, but death-by-exile is now carried by a `death` event with the new `reason: "exiled"` value, and removal from the game is carried by the new `playerRemoved` event. The three are independent so edge cases like a funny Deviant (survives exile) and exiled-but-still-seated travellers are now expressible. The `executed` outcome description was reworded to match: it means "vote-path resolved to execution," not "the player died." Abilities like Devil's Advocate can prevent the death without changing the outcome.
- Added `playerRemoved` event variant for removing a seat from active play. Positions are not reused after removal.
- **Breaking:** `position` is now a stable per-player identifier rather than a seat index. It is assigned at add time and never changes, even when a player physically moves seats mid-game (e.g. a Matron swap). All events that reference players continue to use `position`, and those references remain stable across seat changes. *Seat*, where a player is currently sitting, is a separate concept and is mutated by `playerAdded`, `playerRemoved`, and the new `playerMoved` event. The `players[]` array is in position order.
- **Breaking:** `playerAdded` now requires a `seat` field (1-based) indicating where the new player is inserted. Existing players at that seat or higher shift up by one. Game-start players still have seat == position, but mid-game adds may not.
- `playerRemoved` now specifies that existing players at higher seat numbers shift down by one to close the gap, keeping seats contiguous from 1 to N.
- Added `playerMoved` event variant for recording one or more simultaneous seat swaps (typically a Matron swap). The whole permutation is atomic so consumers don't observe an invalid intermediate state.
- **Breaking:** Renamed top-level `storyteller` to `storytellers` and changed the type from a single `Storyteller` (or null) to an array of `Storyteller` objects, since games commonly have multiple co-Storytellers. Use an empty array when no Storyteller info is available.

## 0.4

- Added optional top-level `storyteller` object with `name` and `id` fields, for carrying the Storyteller's display name and an optional external identifier.

## 0.3

- Added optional `id` field to `PlayerRecord` for carrying an external player identifier.

## 0.2

- Added the `deadVoteUsed` event variant.

## 0.1

- Initial release.
