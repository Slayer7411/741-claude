# ◆ STYX

native macOS digging workstation: bandcamp and the xml collection side by side. Two decks, a decision bar (wishlist, buy, list, gone), a personal library of everything dug, and a download and tag pipeline. The sections below are the app's plan.

## ◆ STATUS

- scaffold, design tokens and settings window: part of the plan's stage 0
- at the last plan update (2026-11-02): read-only reader for the xml collection database and the collection tree in the worker are on a feature branch. A write core for the database exists on another branch, with tests on a synthetic database. The bandcamp web layer and its data work are in the plan's stages 2 and 3.
- 2026-11-10: STYX is to connect to `collection.741` through the 741 client (library, playlists, history). Bandcamp and the existing xml and direct-write paths stay. The STYX plan for that is separate.
- the library of dug items is local data in the STYX database. Purchases and budget data are personal and are not in the shared file by default.

## ◆ WHERE DATA LIVES

- STYX database (sqlite, schema versioned): items, files, decisions, lists, history, purchases, jobs, notes, analysis cache
- settings: a settings store per app (target: `setting` records, app `styx`)
- downloads and previews: folders set in settings
- the xml collection database is read from its path in settings. Writing to it is a plan decision.

