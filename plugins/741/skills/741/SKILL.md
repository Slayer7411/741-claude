---
name: "741"
description: "Knows the 741 tool set and its collection file. Use for anything about .741 files, collection.741, mnemo-agent, the 741 apps (MINOS, ORPHEUS, CHARON, STYX, HELIOS), converting, editing or repairing a 741 collection, playlists, smartlists, cues, beatgrids, tags, sets or app settings in a 741 library, and the nml and xml collection exports and imports."
---

# ◆ 741

741 is a set of macOS music tools that share one collection file, `collection.741`. It holds the library, playlists, sets, history and settings of every app.

## ◆ WHAT TO KNOW FIRST

- one file = library + playlists + sets + history + settings of all 741 apps
- only `mnemo-agent` writes the home folder. Apps and tools talk to it over a local socket.
- the nml collection and xml collection are one-way exports. They are never read back as data.
- status: the format is a draft (schema 1, not released). Fields can still change. Check the app reference for what each app ships today.

## ◆ WHICH REFERENCE TO LOAD

Load only what the task needs.

| task | load |
|---|---|
| file layout, record types, fields, units, dates, ids, merge, smartlist rules, settings registry | `references/format.md` |
| changing data, agent API, change recipes, backups, repair, imports and exports | `references/writing.md` |
| MINOS: features, settings, files, where data lives, ai control (socket, minos-mcp) | `references/minos.md` |
| ORPHEUS: status, services, LAN protocol | `references/orpheus.md` |
| CHARON: recorder, verify, handover playlist format | `references/charon.md` |
| STYX: digging workstation plan and status | `references/styx.md` |
| HELIOS: installer and updater plan | `references/helios.md` |

## ◆ HARD RULES

- never write `collection.741` directly while `mnemo-agent` runs: no editor, no script, no copy over it. Change data with put, get and query over the agent socket, or with the `mnemo` CLI.
- back up before a bulk or manual change: `mnemo backup --keep`. Restore = preview first, then a `pre-restore` backup, then the new generation.
- keep unknown fields and unknown record types exactly as read. Never drop them to clean up.
- write only your own `x.<app>`. Never write `x.agent`. `x.nml`, `x.rb` and `x.minos` come only from an import session.
- measured values are integers. Time is in nanoseconds, tempo in micro-BPM, levels in micro-dB. No floats.
- delete with a tombstone (`del: true`), not by removing a record. Keep `v` unless `del` is true.
- do not write audio file tags yourself. Send `put track` with the tag fields. The agent writes the files and reports `x.agent.tag`.
- secrets never go into the file, a setting or a log.
- if a reader's schema is below the file's `readMin`, open the file read-only.
- never treat an nml or xml export as the source of truth. Imports are one-time or manual, with a preview.
- if the references do not cover a field or rule, say so and ask. Do not invent it.

## ◆ WHEN ASKED TO CHANGE A COLLECTION

1. confirm the agent answers `hello` (or that you are working on a copy or a fixture, never the live file)
2. back up with `mnemo backup --keep`
3. make the change as one batch of put or delete records, with ids and fields that pass the rules in `format.md`
4. verify with `mnemo verify <file>`, then compare record counts with the backup

## ◆ THE APPS IN ONE LINE EACH

- MINOS: native dj software for macos (0.x). Decks, mixer, fx and plugin host, browser, tags, smartlists, analysis, audio edit, set recording. Reads the nml collection today. Library data moves to `collection.741` as planned.
- ORPHEUS: multi-service player (local files, google drive, spotify, tidal, soundcloud, apple music) on desktop and an ios remote. Playlists, history and transfer between services.
- CHARON: native tidal recorder for macos. Captures only the tidal player output, bit-exact, cuts per track, verifies, tags, and registers recordings in the collection.
- STYX: digging workstation for bandcamp and the xml collection. Decks, decision bar, library of everything dug. Connection to `collection.741` is planned.
- HELIOS: installer, updater and setup of the 741 environment. Planned, not built.
- mnemo-agent and the `mnemo` CLI: the single writer of the home folder. Verify, info, backup, prune, restore, import and export, merge.
