# ◆ 741 COLLECTION FORMAT (.741)

Spec: draft v1, 2026-11-03. Schema 1 is not released: fields may still change until release. After release: compatibility rules (end of this file).

## ◆ SCOPE

- one file = library + playlists + sets + history + settings of every 741 app
- read and write: MINOS, ORPHEUS (desktop and iOS), CHARON, STYX
- closed system: 741 apps read only `.741`, never `collection.nml` or xml collection data
- nml collection and xml collection are one-way exports. nml / xml import runs once (migration) or on request
- every app's settings are records in the file
- never in the file: secrets, waveforms, decoded audio, overview caches, logs, write queues, journals, runtime window state, bootstrap paths, STYX purchase ledger

## ◆ FILES

```
<home>/                          default: My Drive/741/
  collection.741                 main file
  .tmp/                          staged snapshot, backup and export files
  exports/collection-741-export.nml
  exports/<xml export file>      name: export.xmlFileName (agent setting)
  backups/<YYYY>/<name>          see BACKUPS
```

- `<home>` is written only by `mnemo-agent`. Apps never write it directly.
- local store: `~/Library/Application Support/741/`. Must be local; cloud paths are rejected.
- local store path and home path are bootstrap values. They live in `agent.json` / the local store, not in the file.

## ◆ CONTAINER

- JSON Lines, UTF-8, LF. One JSON object per line.
- optional gzip for the whole file. Readers detect the gzip magic `1f 8b`. Main file: gzip by default (option). Backups: always gzip.
- gzip: one member only. A multi-member stream is a format error (file not loaded). Writers write one member.
- line 1: header. last line: footer. records in between.
- record order: by type (order of the type table), then by `id` (UTF-16). Writers are deterministic.

### canonical JSON (every line)

- no whitespace
- object keys sorted by UTF-16 code units
- strings escaped exactly like JS `JSON.stringify` (lone surrogates as `\udxxx`)
- integers as plain integers. No floats are written. Number literals in unknown data are written back unchanged.
- byte-identical output is defined on the uncompressed lines. gzip bytes may differ.
- fractional index (order keys): the `fractional-indexing` algorithm, base62 digits

### header

```json
{"fmt":"741","schema":1,"readMin":1,"gen":42,"library":"<uuid>","created":1762128000000,"saved":1762131600000,"device":"<deviceId>","writer":{"app":"minos","version":"0.9.0","schema":1}}
```

- `fmt`: constant `"741"`
- `schema`: format version of the writer
- `readMin`: lowest reader schema that may write the file back
- `gen`: monotonic, +1 per snapshot
- `library`: stable id of this library, never changes
- `created`, `saved`: datetime (unix ms)

### footer

```json
{"end":true,"records":15344,"sha256":"<hex>"}
```

- `sha256` over the uncompressed bytes of all lines before the footer, including their LF.
- missing or wrong footer: the file is incomplete. It is never loaded, never overwritten, and moved aside as `*.unreadable-<stamp>`.

## ◆ RECORD

```json
{"t":"track","id":"<uuid>","h":"<hlc>","v":{...},"fh":{"title":"<hlc>"},"x":{"nml":{...},"rb":{...},"minos":{...}},"del":false}
```

- `t`: record type. `id`: stable id (uuid v4 lowercase, unless the type defines its own id; see IDS).
- `h`: hybrid logical clock of the last change: `<ms 13 digits>.<counter 4 hex>.<deviceId>`. String comparison = clock order.
- `v`: canonical fields of the type.
- `fh`: per-field clocks, for types marked `field` in TYPES. Keys are field names. For `setting`, keys are names inside `value`.
- `x`: extensions. `x.nml` and `x.rb` hold raw source attributes not mapped to `v`. `x.<app>` holds private data of one app. An app writes only its own `x.<app>`. The agent writes `x.agent`.
- `del: true`: tombstone. Kept 365 days after its `h` (option), then dropped at the next snapshot.
- unknown keys at any level, and unknown record types, are kept and written back unchanged.

## ◆ UNITS

- measured values are integers, so every implementation writes identical bytes. Typed integers are safe integers (|n| <= 2^53-1). Larger literals are unknown data and kept verbatim.
- time, position, length: nanoseconds (`ns`). Position counts from sample 0 of the decoded audio after encoder delay is removed.
- tempo: micro-BPM. Example: `130000473` = 130.000473
- level: micro-dB. Example: `-0.000000` = 0
- bitrate: bps (integer). size: bytes (integer). sample rate: Hz (integer)
- confidence: integer 0..1000
- colors: `"#rrggbb"`, lowercase
- key: integer `0..23`. 0 = C, 12 = Cm, chromatic order, minor = +12. The text form is derived, never stored.

## ◆ DATES

| kind | form | fields |
|---|---|---|
| datetime | unix ms UTC, integer | `track.life.added`, `track.life.modified`, `track.file.mtime`, `played.last`, `history.start`, `history.end`, `history.entries[].at`, `track.analysis.at`, `x.agent.tag.at`, `header.created`, `header.saved` |
| date | `"YYYY-MM-DD"` | `set.date` |
| partial date | `"YYYY"`, `"YYYY-MM"` or `"YYYY-MM-DD"`, precision as in the source | `track.tags.released` |

- date-only source into a datetime field: UTC midnight of that date. Independent of the device time zone.
- local-time source (nml MODIFIED_DATE + MODIFIED_TIME, MINOS history start date + time): local date and time in `device.tz`. A nonexistent local time takes the offset before the gap. An ambiguous local time takes the earlier instant.
- `device.tz`: IANA name, required on the device record.

## ◆ IDS

- charset: app `[a-z]+`; setting group `[A-Za-z][A-Za-z0-9_-]*`; setting name `[A-Za-z0-9._-]+`; deviceId `[A-Za-z0-9._-]{1,64}`
- `/` and `:` are never part of an app, group, name or deviceId.
- `:` appears only in the scope token `d:<deviceId>` and as the separator of stream ids (service prefix, split at the first `:`).
- `/` separates the parts of a setting id only. No escaping. Ids outside the charset are rejected at write.
- identity is the (type, id) pair. Strings are compared as raw UTF-16 units. No Unicode normalization: ids that differ only by normalization are distinct.

## ◆ TYPES

| t | id | merge | fields (v) |
|---|---|---|---|
| `library` | header `library` | record | name, musicRoots (ordered root labels), tagSeparators (absent = space), `x.nml` (library level) |
| `root` | label | record | name. Path per device is in `device` |
| `device` | deviceId | record | name, platform, tz, rootPaths `{label: absolute path}`, volumes `{label: VOLUME name}` (optional) |
| `track` | uuid | field | see TRACK |
| `cue` | uuid | record | track, kind, slot, pos, len, name, color, order |
| `grid` | track id | record | markers `[{pos, beat, bpm, rb}]`, source, metro |
| `playlist` | uuid | field | name, parent, order, kind, color, role, rule, set |
| `item` | uuid | record | playlist, track or stream, order, note, set |
| `history` | uuid | record | device, app, start, end, entries |
| `played` | track id | counter | counts `{deviceId: int}`, last |
| `cut` | uuid | record | track, start, end (ns), name |
| `edit` | track id | record | trims `[{from, to}]` (ns) |
| `stream` | `<service>:<id>` | field | service, id, title, artist, duration (ns), url |
| `recording` | uuid | record | service, id, status, track |
| `setting` | `<app>/<scope>/<group>` | field | app, scope, group, value, appSchema |

Merge kinds: `record` (higher `h` wins), `field` (per field by `fh`), `counter` (see MERGE).

### track

- flat: every track field below is a top-level key of `v`. Objects are only `analysis`, `tagsRaw`, and the arrays `userTags` and `streams`. The groups file / tags / dj / life are documentation only.
- file: `root`, `rel` (path under the root, `/` separated, NFC), `size` (bytes), `mtime` (datetime), `codec` (lowercase: `mp3`, `m4a`, `wav`, `aiff`, `flac`, ...), `sampleRate` (Hz), `bitrate` (bps), `duration` (ns), `fp`
- `fp` = `<sha1 hex of the first 64 KiB>-<size>-<duration ns>`. Used for rename and move detection.
- unknown file fields are omitted, not written as null.
- `root` not present on this device: the track is shown offline, the record is unchanged.
- tags: `title, artist, album, albumArtist, remixer, producer, composer, mix, label, catalog, genre, grouping, comment, comment2, lyrics, released` (partial date), `trackNo, trackOf, discNo, discOf` (digit text, or text such as `A1`), `isrc` (strings). `year` (integer).
- `tagsRaw`: object of unknown file tags. String values. Keys are ASCII-lowercased. Equal keys: values joined with LF in source order. Written by the agent from the file, never removed.
- dj: `bpm` (micro-BPM, effective), `bpmLocked` (bool), `key` (0..23, effective), `rating` (0..5), `color` (`#rrggbb`, optional)
- `userTags`: array of strings. Canonical user tags. Import splits `comment2` at the library `tagSeparators` (each character separates; empty = space), trims, drops empty, keeps the first occurrence per comparison form (case, diacritics and width folded), keeps order. Export: `comment2` keeps its raw text while its split still equals `userTags`; otherwise `userTags` is written joined with the first separator. Editing `userTags` never changes `comment2`.
- life: `added` (datetime), `modified` (datetime), `cover` (sha256 hex of the image bytes)
- `analysis` (optional, all fields optional): `{version, at, source, tempo: {bpm, conf}, key: {value, conf}, loudness: {peak, perceived}, grid: {anchor}}`.
  - `version`: analyzer version (integer). `at`: datetime. `source`: app that analysed, or `nml`
  - `tempo.bpm` micro-BPM, `key.value` 0..23, `conf` 0..1000
  - `loudness.peak`: sample peak, micro-dB. `loudness.perceived`: integrated loudness, micro-dB
  - `grid.anchor`: first downbeat of the analysis grid (ns). Kept while a manual grid exists. Used to restore the analysis grid on reset. Without an anchor the grid is kept and the track is analysed again.
  - `bpm` and `key` are the effective values. User edits do not overwrite `analysis`.
- links: `streams` (stream ids), `origin` (recording id, optional)
- `x.nml`: nml attributes without a canonical field, plus overlay values (COLOR, KEY, LOCK, VOLUME, VOLUMEID, COVERARTID, FILESIZE, RANKING, dupKey, noFloat)
  - overlay rule: an overlay value is written on export only while it still derives to the current canonical value. Otherwise the canonical value is written.
- `x.rb`: xml collection attributes without a canonical field (TrackID), same overlay rule
- `x.minos`: `ripcheck` (file check findings), `audioedit` (audio edit state), `manual` (`{bpm, key}` booleans: hand-set values that analysis does not overwrite). EDL trims are `edit` records; marks are `cue` records of kind `mark`.

### cue

- kind: `cue | fadein | fadeout | load | loop | mark`. Unknown kinds are kept verbatim and not exported.
- `mark`: audio edit marks. slot `-1`, len `0`. Not exported to nml or xml collection.
- slot: `-1` = memory cue, `0..7` = hot cue A..H. `mark`: `-1`.
- pos (ns), len (ns; `0` except loops), name (`""` = unnamed), color (`#rrggbb`, optional)
- order: fractional index string, per track. Import keeps source order. Export sorts by `order`.

### grid

- id = track id. `markers`: list of `{pos (ns), beat (int, 1 = bar start), bpm (micro-BPM), rb (optional raw values)}`
- marker 0 has no `bpm`: its tempo is the track `bpm`. Markers after the first require `bpm`.
- metro: `4/4` by default
- `source`: string. MINOS writes `AutoGrid` for analysis grids and `Grid` for hand edits. A grid whose source is not `AutoGrid` is manual.
- grid lock: `bpmLocked` on the track only

### playlist

- `kind`: `folder | playlist | smart | set`
- `order`: fractional index string (base62). Concurrent inserts merge without renumbering.
- `parent`: playlist id, or absent (root)
- `role`: `preparation` | `archive` | absent. A new `preparation` sets the previous one to `archive` in the same write. Shared by MINOS and ORPHEUS.
- `rule`: smartlist rule (see RULE). Only for `kind: smart`.
- `set` (only for `kind: set`): `{date (date), venue (text), targetLength (ns), notes (text), status (draft | final | performed), recording (recording id, optional)}`

### item

- `playlist`, then `track` (uuid) or `stream` (stream id). `order` (fractional index, ties by id). `note` (text).
- a track that is not on this device is allowed. Its root resolves through `device.rootPaths`.
- `set` (set items only):
  - `in`, `out`: `{"cue": <cue id>}` or `{"pos": <ns>}`
  - `mixBeats`: integer 0..64
  - `transition`: `{type: cut | blend | echo | filter | loop | other, beats: int, note: text}`. Describes the step from the previous item into this one.
  - `energy`: integer 0..10

### history

- one record per app session (app launch to quit). `start`: first play. `end`: end of the last entry, updated per entry. Both datetime (ms).
- `device`, `app`: device id, app id
- `entries`: one per play, in play order: `{track | stream, at (datetime ms, play start), from (ns), to (ns), deck (int, optional)}`

### played

- id = track id. Single source of play counts. `track` has no play count and no last-played field.
- `counts`: `{deviceId: int}`. Each device writes only its own key. Count = sum of the values.
- `last`: datetime (ms), latest play start of any device.

### cut, edit

- `cut`: id = id of the cut file's own track record. `track` (source), `start`, `end` (ns), `name`.
- `edit`: id = track id. `trims`: list of `{from, to}` (ns). The MINOS edit list.

### stream

- id `<service>:<id>`. Split at the first `:`. Service registry: `tidal`, `soundcloud`, `spotify`, `apple`, `gdrive`, `bandcamp`. Unknown services are kept verbatim.
- `service`, `id` (provider id, may contain `:`), `title`, `artist`, `duration` (ns), `url` (optional, not secret, may expire)
- Bandcamp before download: a `stream` record with `x.styx` data.

### recording

- uuid. `service`, `id` (provider id), `status` (`done | failed | cancelled`), `track` (track id, set when `done`)
- written only at a final status. Progress is local.

### setting

- id `<app>/<scope>/<group>`. `scope`: `g` (global) or `d:<deviceId>` (device). The same token appears in the id and in the field.
- one record per (app, scope, group). Each name is written in its registry scope.
- `value`: object `{<name>: <value>}`. Value types: `null` (app default), boolean, safe integer, string, array, object. No floats: store integers in the unit of the name (percent, micro, milli). No binary, no base64.
- size: canonical `value` at most 64 KiB (UTF-8). Larger writes are rejected. Large data stays local.
- merge: per name. `fh` keys are the names inside `value`.
- `appSchema`: integer. The app migrates its own names.
- `v.app` = the app of the id. Owner = the sending app, or `mnemo` (agent options).

### setting registry (group, scope)

| app | group | names | scope |
|---|---|---|---|
| minos | audio | deviceUID, sampleRate, bufferSize, internalPair, deckPairs, previewPair, mixerMode, parallelDecks, masterVolume, record.inputUID, record.inputFirstChannel | d |
| minos | ui | pair | d |
| minos | pairing | peers (ids and names only), peerID, peerName. Pairing keys stay in the keychain | d |
| minos | donate | donated, donateCode, donateLast, donateHint | d |
| minos | session | decks, fx, clock, browser | d |
| minos | ui | remaining UI preferences | g |
| minos | controller | controller preferences | g |
| minos | fx | engine preferences (non-audio); plugin state stays local | g |
| minos | record | recording preferences except device names (`record.<field>`) | g |
| minos | shortcuts | bindings | g |
| minos | columns | layout (column widths and order) | g |
| minos | wave | presets | g |
| minos | keyprofiles | profiles | g |
| charon | output | outputFolder, libraries, ffmpegPath | d |
| charon | rate | followSourceRate, restoreDeviceRate, rateFallback | d |
| charon | rateRestore | uid (hardware uid), rate | d |
| charon | debug | keepRawCapture, diagLog | d |
| charon | recording | filePattern, skipExisting, quality, format, mp3Bitrate, aacBitrate, embedCover, coverSize, writeCoverFile, trimDigitalSilence, flagOffGrid, pauseBetweenTracksMs, retryOnGap, muteWhileRecording, hideTidalWindow, startTidalWithCharon, language, convertJobs | g |
| styx | xml collection | dbPath, xmlPath, writeMode, backupsKeep | d |
| styx | xml collection | playlistFolder, xmlDebounceMs, xmlIncludePreviews | g |
| styx | audio | outputDevice, bufferSize, cueDevice, cueMix | d |
| styx | midi | device, enabled | d |
| styx | paths | dataFolder, libraryFolder, previewsFolder, analysisCacheFolder, charonInbox | d |
| styx | runtime | heartbeatMs, workerConcurrency, restartBackoffMaxMs, logLevel | d |
| styx | layout | layout, restoreLayout | d |
| styx | privacy | all names | d |
| styx | general | onboardingDone | d |
| styx | decks | count, keylock, pitchRange, keyNotation, crossfaderCurve, heardSeconds, heardPercent, wavePreset, phraseLength, phraseBands, barMarkers, gridLines, spacePlaysA, timeDisplay | g |
| styx | downloads | format, formatOrder, convertTo, writeTags, artwork, autoDownload, keepOriginal, ownedCheck, deletePreviewAfterBuy | g |
| styx | bandcamp | all names except the local sync timestamp | g |
| styx | decisions | budget names | g |
| orpheus | device | deviceName, outputDeviceId, lanEnabled, lanPort, lanPauseLocalWhenRemote, sonos.*, updates.* | d |
| orpheus | providers | showPairMark, providerClientIds, serviceMode, gdriveMount.enabled | g |
| orpheus | mount | gdriveMount.path | d |
| orpheus | library | minos.enabled, minos.refreshMinutes, minos.tagSeparators, collection.keyNotation | g |
| orpheus | collection | collection.home | d |
| orpheus | playback | playback.maxBoost, playback.holdPresses, playback.snapWidth, playback.holdWindowMs, playback.boostZone | g |
| orpheus | player | playback.volume | d |
| orpheus | appearance | appearance.textSize | g |
| orpheus | search | searchTimeouts.webMs, searchTimeouts.apiMs | g |
| orpheus | sources | sources.rememberFolders, sources.expandedFolders | g |
| orpheus | airplay | enabled, name, latencyMs, target, passwordSet (flag; password in the secret store) | d |
| mnemo | snapshot, backup, export (global), export.folder (device), tags | agent options, see WRITING | g / d |

Not settings (local only): STYX sync timestamp, STYX web tab state, Orpheus queue (paths, index, position), Orpheus paired-device token hashes, journals, caches.

## ◆ RULE (smartlist)

A `playlist` with `kind: smart` has a `rule` JSON object:

```json
{"where":{"all":[{"f":"genre","op":"contains","v":"techno"},{"any":[{"f":"rating","op":"ge","v":4},{"f":"bpm","op":"lt","v":100}]},{"not":{"f":"artist","op":"equals","v":"zed"}}]},"sort":[{"f":"bpm","dir":"desc"}],"limit":50}
```

- keys: `where` (expression; absent = no expression), `sort` (list of `{f, dir}`, `dir` is `asc` by default or `desc`), `limit` (integer >= 0), `raw` (nml query text with no parsed form). Unknown keys are kept.
- node: `{"all":[...]}` (empty = true), `{"any":[...]}` (empty = false), `{"not": node}`, or leaf `{"f","op","v"}` (no `v` for `isEmpty`).
- operators: `equals notEquals contains notContains starts ends lt le gt ge between inLastDays isEmpty containsAll containsAny`
- fields by kind:
  - text: `title artist album albumArtist remixer producer composer mix label catalog genre grouping comment comment2 lyrics isrc color keyText root rel codec filepath filename`
  - number text: `trackNo trackOf discNo discOf`
  - number: `bpm key rating playCount bitrate duration size sampleRate year`
  - date: `added modified lastPlayed released`
  - bool: `played`
  - state: `fileState` (`"present"` / `"missing"`)
  - array: `userTags` (`contains`, `notContains`, `containsAll`, `containsAny`, `isEmpty`)
- virtual fields: `playCount` = sum of `played.counts`. `lastPlayed` = `played.last`. `played` = playCount > 0. `keyText` = text derived from `key`. `filepath` and `fileState` are evaluated per device. `filename` = last segment of `rel`.
- text compare: NFD, combining marks removed, lowercased. No other folding (`ß` is not `ss`).
- number text: both sides pure ASCII digits compare by integer value. Otherwise text compare; digit text sorts before other text.
- units in rules: bpm in BPM, duration in seconds, bitrate in kbps, size in bytes. Scaled values round half away from zero.
- missing values: positive comparisons are false, `notEquals` is true, `isEmpty` is true. `rating` and `playCount` absent = 0. `key` absent = -1.
- sort: keys in order, then ascending id. Missing values last in both directions. `limit` applies after sort.
- unknown rule (unknown field or operator, bad value, missing context, `raw` only): the result has no ids. The list keeps its stored items. An unknown rule never empties a list.
- analysis values, tags lists, streams and origin are not rule fields.

## ◆ TAG WRITES

- the agent alone writes audio file tags. Apps change tags with `put track` (tag fields).
- when a `put` changes a tag field, the agent writes the file, recomputes `size`, `mtime` and `fp`, and writes the new values in the same snapshot.
- result: `x.agent.tag = {h (h of the applied put), at (datetime), ok (bool), error (text, optional)}`
- failed write: `ok: false`. The record keeps the requested tags and is retried at the next start.
- `tagsRaw` and unknown file tags are preserved by agent writes.
- new files: the encoder writes the tags before the first `put`. After that, all tag writes go through the agent.

## ◆ SECRETS

Never in the `.741` file and never in a `setting` value.

| secret | store |
|---|---|
| MINOS pairing keys | macOS keychain |
| TIDAL token | keychain |
| TIDAL web session | WebKit session store |
| Bandcamp cookies | HTTP cookie storage |
| ORPHEUS provider OAuth tokens | Electron safeStorage, one file per key |
| ORPHEUS phone pairing token | keychain |
| STYX xml collection database passphrase | not in the file |

## ◆ MERGE

- per record: higher `h` wins. Field types (`track`, `playlist`, `stream`, `setting`): per field by `fh`. Fallback clock = record `h`. Ties by canonical bytes.
- `counter` (`played`): `counts` per deviceId take the max. `last` takes the max. Each device writes only its own key, so no increment is lost.
- tombstone wins over an older live record and loses to a newer one.
- unknown records: higher `h` wins, otherwise union.
- items: union. Order by fractional index, ties by id.
- `x` and unknown top-level keys: merged by key, newer wins conflicts.
- result is written with `gen = max(local, remote) + 1`.
- clocks: hybrid logical clock. The wall clock never moves backwards per device.

## ◆ EXPORT

- on quit of a 741 app: nml collection and xml collection exports (default on, per target on/off)
- fixed names, overwritten each time. Folder and names are agent options.
- written atomically (temp file and rename). Quit waits at most the export timeout (default 10 s). An unfinished export is completed by the agent.
- never written over a live collection of another app unless that option is set explicitly.
- a track whose root is not in the exporting device's `rootPaths` is exported with `x.nml` LOCATION if present, otherwise skipped with a warning.
- nml collection: `NML VERSION="19"`, collection, playlists and smartlists, all mapped fields plus the `x.nml` overlay. Cue times in ms with 6 decimals (exact ns). Grid as a single type-4 cue at marker 0. `mark` cues are not exported.
- xml collection: `DJ_PLAYLISTS Version="1.0.0"`, positions per grid marker, hot cues and memory cues. Smartlists are exported as static playlists of their evaluated items. `mark` cues are not exported.
- offsets: `positionOffsetNs[codec]` is applied to cue and grid positions on export and subtracted on import. Default 0.

## ◆ IMPORT

- nml collection and xml collection: at first start (migration) and on manual request only. No automatic re-import.
- manual import merges with the source file's modification time, and shows a preview of changed tracks first.
- exports are never read back by the apps.
- roots: a track's root is the longest device `rootPaths` value that prefixes its directory. No match: a new root labelled with the volume name.
- duplicates (same path or key): the first is the track. Each further entry is a separate track with `x.nml.dupKey`. Ids are UUID v5 of the key; the n-th repeat uses UUID v5 of `<key>#n`. No automatic merge (`mnemo merge`, on request).
- nml import: `TYPE` maps to cue kinds (grid anchor becomes the `grid` record). `RANKING` maps to `rating` as `round(RANKING / 51)` clamped to 0..5. Raw values that do not convert are kept in `x.nml`.
- xml import: ids are UUID v5 of the absolute path. The n-th repeat of the same path uses `<path>#n`.

## ◆ WRITERS

- every app writes (put and delete) through the agent API. The agent alone writes `<home>`.
- snapshot: temp file in `<home>/.tmp/`, fsync, rename. Debounce 45 s after the last change (option 5–600 s).
- local journal: append-only, fsync per batch, replayed on start.
- an app with no agent reachable writes its local outbox (see WRITING).

## ◆ BACKUPS

- name: `<YYYY-MM-DD>T<HH>Z.g<gen>.<deviceId>.741.gz`, path `backups/<YYYY>/<name>`. All times UTC.
- tagged name: `<YYYY-MM-DD>T<HH>Z.g<gen>.<deviceId>.<tag>.741.gz`. Tags: `pre-import`, `pre-migrate-<from>-<to>`, `pre-restore`, `keep` (manual).
- hourly: at the full hour, only if `gen` changed since the last backup (interval option, default 60 min).
- tagged backups are never pruned.
- retention (defaults, all options):
  - every hourly (untagged) backup for 7 days
  - newest per ISO week for 4 more weeks
  - newest per month for 12 months
  - newest per year, kept forever
  - optional: newest per day for N days (default off)
- pruning is deterministic from the file list. Two devices pruning at once delete the same files.
- never prune when the newest backup or the main file fails the footer check.
- restore: preview (read-only open), then a `pre-restore` backup, then a new `gen`.

## ◆ COMPATIBILITY

- reader `schema >= file.schema`: normal.
- `file.readMin <= reader.schema < file.schema`: read and write. Unknown fields and types are kept verbatim. Banner "newer version".
- `reader.schema < file.readMin`: read-only. Banner "update required".
- within a schema: only additive changes. No rename, no reuse of a field name, no unit change.
- breaking change: new schema, a step migration `N -> N+1` in every implementation, and `readMin` raised.
- before any migration: backup tagged `pre-migrate-<from>-<to>` (never pruned).
- `fixtures/` holds a file of every released schema. Every implementation must load them.
- until schema 1 is released, its fields may change in the spec.
