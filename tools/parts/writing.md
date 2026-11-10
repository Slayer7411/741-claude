# ◆ CHANGING A 741 COLLECTION SAFELY

Applies to every change: editing data, fixing a file, importing, migrating. Format details: `format.md`.

## ◆ BEFORE YOU CHANGE ANYTHING

- never write `collection.741` directly while `mnemo-agent` runs. No editor, no script, no copy over the file.
- changes go through the agent (put, get, query over the socket) or through the `mnemo` CLI.
- back up first: `mnemo backup --keep` writes a `keep` backup. Tagged backups are never pruned.
- work on a copy or a fixture when you test. Use an isolated home and local store for tests (own `--socket-dir`, own `agent.json`).

## ◆ CHECK THE AGENT

- socket: `<socket-dir>/agent.sock` (mode 0600). Default socket dir: `$TMPDIR/741`.
- single instance: `flock` on `<socket-dir>/agent.lock`. A second start exits with status 1.
- first request must be `hello`. Its reply gives `home`, `localStore`, `protocol`, `schema`, `gen`.
- protocol: major must match (else `unsupportedVersion`). Minor versions are additive.
- if no agent answers: the change waits in the app outbox (below). Do not write the file instead.

## ◆ SOCKET AND FRAMES

- frame: 12-byte header, little endian: `MIPC` (4 bytes), type `u16`, version `u16`, length `u32`, then a JSON object payload (max 1 MiB; larger frames close the connection).
- version `u16` = `major << 8 | minor`. Protocol 1.4 = `0x0104`.
- request: `{"id": n, ...}`. reply: `{"id": n, "result": {...}}`. error: `{"id": n, "error": {"code", "message"}}`. event: `{"records": [...]}` with no id.
- message types: hello 1, put 2, get 3, query 4, subscribe 5, status 6, flush 7, appQuit 8, ping 9, loaded 10. Replies: reply 100, error 101, event 102.
- hello request: `app`, `appVersion`, `schema`, `deviceId`. Any request before hello (except ping) fails with `notHello`.
- error codes: `badRequest`, `unknownType`, `notHello`, `forbidden`, `invalidRecord`, `unknownRecordType`, `unsupportedVersion`, `tooLarge`, `internalError`, `unknownOperator`, `badFilter`.
- get: `type`, `ids`, `cursor`. query: `type`, `filter`, `cursor`. Replies hold `records`, `errors`, and `more` when a further page exists. Send the same request with `cursor: more`.
- query filter operators: `eq`, `contains` (array element), `in`, `prefix`, `exists`, `all` (AND). Top level also takes `deleted` (bool) and `limit` (int). Paths use dots into objects of `v`.

## ◆ PUT RULES

- the agent stamps `h` and, for `track`, `playlist`, `stream` and `setting`, `fh`. Client `h` and `fh` are ignored.
- a batch is all or nothing. The journal is fsynced before the reply.
- `at` (optional, unix ms): the real edit time of an offline change. With `at`, `h` = `at.0000.<deviceId>`, so the order of offline edits is kept. Replays with the same `at` give the same stamp.
- write rules:
  - `x` keys: only `x.<own app>`. `x.nml`, `x.rb`, `x.minos` only from the import session. `x.agent` never.
  - `setting`: the id's app must be the sending app (or `mnemo` for agent options).
  - `v` is required unless `del` is true.
- the agent does not write `played` (only the importers, and only their own key).

## ◆ CHANGE RECIPES

- rename a track: `put track` with the changed `title` (only the changed fields need values; the record keeps the rest). Tag fields go to the file through the agent.
- set a cue: `put cue` with `track`, `kind`, `slot`, `pos` (ns), `len`, `name`, `order`. Hot cue `slot` 0..7, memory cue `-1`.
- add a playlist item: `put item` with `playlist`, `track`, and an `order` between its neighbours (fractional index). Do not renumber other items.
- delete a record: `put` with `del: true`. Do not remove the line from the file.
- smartlist: `put playlist` with `kind: smart` and a `rule` JSON (see format.md RULE). A rule the apps cannot evaluate is kept as unknown; it does not empty the list.
- settings: `put setting` with id `<app>/<scope>/<group>`, `value` as an object of names. Use `null` for app default. Integers only.
- a set: `playlist` `kind: set` with `set` fields; `item` with `set` fields. Keep `mixBeats` 0..64, `energy` 0..10.

## ◆ TAGS

- send `put track` with the tag fields. Do not write audio files yourself.
- the agent writes the file, recomputes `fp`, `size` and `mtime`, and sets `x.agent.tag`.
- check `x.agent.tag.ok`. On `false`, the error text is in `x.agent.tag.error` and in the agent `tags.log`. The write is retried at the next start.
- the agent queues a file for writing only after the put debounce. It touches at most `tags.perMinute` files per minute.
- a file or track that a player app has loaded is deferred (loaded guard). Writes wait while a player app is connected and has not yet reported its loaded files.

## ◆ OFFLINE AND OUTBOX

- client outbox: `<local store>/outbox/<app>.jsonl`. Every put is appended and fsynced before it is sent.
- an entry is removed when the agent answers it (accepted, or rejected for good: `forbidden`, `invalidRecord`, `unknownRecordType`, `badRequest`, `tooLarge`).
- other failures (disconnect, `internalError`) keep the entry. It is replayed in file order after start and after every reconnect.
- size cap: 50 MB of pending entries by default. Over the cap, `outboxFull` at once; nothing is written.
- Swift clients: call `flushOutbox()` before the process exits. TypeScript clients: `put` appends before it returns.
- live paths: no agent call on the audio thread, the control queue or the main thread. Playback reads come from an in-memory mirror. Writes go to one utility-priority queue and are never awaited there.

## ◆ BACKUPS AND RESTORE

- automatic: at each full hour, if `gen` changed. Then retention runs on the file list.
- manual: `mnemo backup --keep`.
- restore: preview first (read-only open, library id must match), then a `pre-restore` backup of the current file, then a new `gen` (max of current and backup gen, plus 1).
- an unreadable main file is moved aside, never overwritten.
- retention preview: `mnemo prune --dry-run`. Run it before a real prune.

## ◆ IMPORT, EXPORT, MERGE

- import: `mnemo import-nml`, `mnemo import-rb` (xml collection). Once at migration, or on request. Show the preview; never run an import while a deck plays.
- export: `mnemo export-nml`, `mnemo export-rb`. Quit exports run automatically. Exports are never read back.
- duplicates: `mnemo merge` on request. Imports never merge duplicates automatically.
- verify: `mnemo verify <file>` (0 ok, 1 usage, 2 format error). `mnemo info` shows header and counts. `mnemo canon` writes canonical bytes.

## ◆ REPAIR

- missing or wrong footer: the file is incomplete. It is moved aside as `*.unreadable-<stamp>` and never overwritten. Restore from a backup.
- footer ok, but a record is wrong: fix it with `put` (new `h`), not by editing the line.
- unknown fields and record types: keep them. Do not drop them to "clean up".
- duplicate `(type, id)` records: rejected on read. Find the source (import, two writers) before you change anything.
- schema newer than your reader: read-only. Do not write.
- after any repair: `mnemo verify`, then `mnemo info`, then compare counts with the backup.

## ◆ AGENT CONFIG

File: `<local store>/agent.json` (default `~/Library/Application Support/741/agent.json`). Canonical JSON, one line. Missing keys take the default. Changes that are stored as `setting` records of app `mnemo` apply live.

| key | default | note |
|---|---|---|
| `home` | Drive `My Drive/741` | required in a hand-written file |
| `localStore` | `~/Library/Application Support/741` | must be local |
| `device` | sanitized host name | `[A-Za-z0-9_-]`, 1 to 64 chars |
| `library` | new uuid | written on first start |
| `snapshot.debounceSec` | 45 | 5 to 600 |
| `backup.intervalMin` | 60 | |
| `backup.hourlyDays` | 7 | |
| `backup.dailyDays` | 0 | 0 = off |
| `backup.weeklyWeeks` | 4 | |
| `backup.monthlyMonths` | 12 | yearly tier is kept forever |
| `gzip` | true | main file |
| `export.nml`, `export.xml` | true | per target |
| `export.nmlFileName`, `export.xmlFileName` | `collection-741-export.nml`, default xml export name | file name only |
| `export.folder` | `<home>/exports` | |
| `export.timeoutSec` | 10 | 1 to 60 |
| `export.onAnyQuit` | false | |
| `tags.write` | true | |
| `tags.writeTagsToCloudFiles` | false | |
| `tags.perMinute` | 20 | 1 to 600 |
| `tags.id3Version` | 4 | 3 or 4 |
| `tags.preserveMtime` | false | |
| `tags.playerGraceMin` | 10 | 0 to 1440 |

Not settings (agent.json only): `home`, `localStore`, `device`, `library`.

## ◆ NOT IMPLEMENTED (agent, per its spec)

- Ogg Vorbis and Opus tags (read and write)
- agent cover writes from the local image cache
- journal archive for devices older than tombstone retention
- exports after each snapshot
- Windows, Linux and iOS transports
