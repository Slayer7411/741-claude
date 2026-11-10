# ◆ ORPHEUS

multi-service player. desktop (electron: macos, windows, linux) and ios (swiftui remote control for paired desktop players, plus local playback of files served by a desktop player). One queue, playlists and history across local files and the services below.

## ◆ STATUS

- local files, drag and drop, folders: implemented, covered by end-to-end tests
- queue, history (appended after 30 s or at track end), playlists, session: implemented. The session starts empty on every launch and is never restored.
- google drive: library and playback implemented, tested with fakes, needs an account. Files named `.741` under the music root are read as read-only records (house files). They are not written back.
- spotify: library and playback implemented, tested with fakes. Needs a premium account. Playback needs the Widevine build.
- apple music: library and playback implemented, tested with fakes. Needs a developer account.
- soundcloud: library and hls playback implemented, tested with fakes.
- tidal: login, library, search, playlist reads and playlist writes (official API mode) implemented. Endpoints are not yet verified against the live API. Playback in official API mode is covered by fake tests only. Playback in web mode is not implemented and fails with an explicit error.
- transfer of tracks between services: implemented. Matches by ISRC, then by text, with a score and an ask dialog.
- output picker and local outputs: implemented
- AirPlay receiver (AirPlay 1, audio only, desktop): implemented
- phone server (desktop serves files imported on the phone over http, for a room or a sonos target): implemented
- LAN server on desktop (discovery, websocket, pairing): not implemented. The desktop does not serve the LAN protocol yet, so iOS remote control does not reach a desktop yet. iOS discovery and client are implemented against fakes.
- iOS app: implemented against fakes. Not run on a device for the current branch.
- windows and linux packages: defined in CI, not built or run yet.

## ◆ DATA AND SECRETS (today and target)

- desktop state (playlists, smartlists, settings) and history: files in the app's user-data folder, written atomically. Target: `playlist`, `item`, `history` and `setting` records in `collection.741`.
- artwork and waveform caches: local, safe to delete.
- secrets (OAuth tokens, user tokens): Electron safeStorage, never in json and never logged. The iOS pairing token is in the keychain.
- Target per format.md: settings are `setting` records with app `orpheus`. The Orpheus queue and the LAN pairing token hashes stay local.

## ◆ SERVICE SETUP

Service client ids come from settings or the desktop `.env` file. A missing id shows the service as `setup` in the services view. Environment names and registration steps are in the section below.

