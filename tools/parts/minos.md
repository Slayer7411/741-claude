# ◆ MINOS

native dj software for macos (0.x, beta). 4 decks, internal or external mixer, own browser, tags, smartlists, analysis, audio edit, recording, fx and plugin host. The sections below are the app's feature, settings and library docs.

## ◆ STATUS

- 0.x: things still change.
- the collection today is the nml collection. The newest `collection.nml` in the library folder is loaded at start. Other collections open with the open-file command.
- write-back to the nml collection is opt-in and off by default. With it on, changes are written in one pass about 1.5 s after the last apply, after a backup. Only the edited entries change. The write is checked and replaces the file atomically. It does not run while the library app is running.
- write-back maps: metadata to entry attributes; own analysis to tempo, loudness, key and the first grid marker; hot cues 1-8 to cue entries; hand grid to tempo and the first grid marker; renamed files to the location; cut copies and tool loops to new entries; preparation and history to playlists in the `MINOS-741` folder; own playlists to the `MINOS-741` folder.
- changes that stay in MINOS-741 only: trim (cut in / out), active loops, undo.
- conflict: when an entry changed both in MINOS and in the file, the file wins for the whole entry. MINOS keeps its open changes in conflict files in the data folder.
- target: library data moves into `collection.741` through `mnemo-agent`. See format.md for the record types MINOS will use (track, cue, grid, edit, cut, playlist, item, history, played, setting). Until a release ships this, the data folder below is the source of truth.

## ◆ FIRST START

1. close other apps that use the controller (the controller talks to one app at a time)
2. start MINOS. It loads the newest `collection.nml` in the library folder. Other collections: open-file command.
3. settings, audio: pick the output device and the mode. Internal = mixer in the app, master on one pair. External = each deck on its own pair of the desk.
4. settings, controller: check which controller drives which decks. The side is saved per device serial number.
5. load a track: double click (focused deck, 1-4 keys), right click (deck A-D), or the load button on the controller.

Nothing is analysed on start. Tracks without an analysis play, but have no grid, so they do not sync. Right click, then Analyse, computes tempo, grid, key and loudness.

## ◆ FILES AND WHERE DATA LIVES (today)

| what | where |
|---|---|
| collection (read; write only on request) | newest `collection.nml` in the library folder |
| collection backups (write-back on) | `Backup MINOS-741` next to `collection.nml` |
| track metadata edits | `~/Library/Application Support/MINOS-741/metadata.json` |
| settings | user defaults, key `prefs.v1` |
| session (decks, fx, clock, browser) | `~/Library/Application Support/MINOS-741/session.json` |
| hot cue, grid and cut edits | `~/Library/Application Support/MINOS-741/edits.json` |
| cut files | `~/Library/Application Support/MINOS-741/cuts.json` (files next to their originals) |
| browser previews | `~/Library/Application Support/MINOS-741/overviews/overviews.bin` |
| own analysis (tempo, grid, key, loudness) | `~/Library/Application Support/MINOS-741/analysis.json` |
| history | `~/Library/Application Support/MINOS-741/History` |
| preparation archive | `~/Library/Application Support/MINOS-741/Preparation` |
| cover thumbnails (cache, safe to delete) | `~/Library/Caches/MINOS-741/artwork` |
| set recordings and tracklists | `~/Music/MINOS-741/Recordings` (setting) |

Backup = copy the `MINOS-741` folder.

## ◆ RECORDING (summary)

- one click on the record control starts and stops a set recording. Source: internal master, or the desk input in external mode.
- files: lossless m4a, wav or aiff at 24 bit, named by date and time. A tracklist (`.txt`) and a cue sheet (`.cue`) are written next to the file, from the play history.
- the file is closed cleanly on quit, device loss or sleep. A device coming back does not resume the recording.

