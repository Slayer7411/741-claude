
## ◆ 741-PLAYLIST v1 (STYX to Charon handover)

A playlist from STYX (bandcamp digging) for Charon. STYX writes the file into the inbox folder. Charon reads it, shows the titles for review, and on request creates a tidal playlist.

- folder: `~/Library/Application Support/Charon-741/inbox/`
- name: `741-playlist-<app>-<timestamp>.json`
- write atomically: write `<name>.json.tmp` (or a dot file) first, then rename. Charon reads only `*.json` directly in `inbox/`, and skips files younger than 2 s.
- after reading: `inbox/done/` on success, `inbox/failed/` on error (with `<name>.reason.txt` next to it).

Format:

```json
{
  "format": "741-playlist",
  "version": 1,
  "source": { "app": "styx", "version": "0.4.0", "exported": "2026-11-02T10:15:00Z" },
  "created": "2026-11-02T10:15:00Z",
  "playlist": { "kind": "rb_playlist", "id": "rb-17", "name": "Dubstep Digs" },
  "tracks": [
    {
      "position": 1,
      "id": "4411022",
      "artist": "Koan Sound",
      "title": "Sun Cave (Original Mix)",
      "version": null,
      "album": "Sun Cave EP",
      "label": "Kalahari",
      "duration": 312.4,
      "isrc": "GBXYZ2600001",
      "source": { "bandcamp": { "trackId": 4411022, "pageUrl": "https://example.bandcamp.com/album/sun-cave-ep", "trackIndex": 1 } },
      "localFile": null
    }
  ]
}
```

| field | required | meaning |
|---|---|---|
| `format` | yes | `741-playlist` |
| `version` | yes | `1`. Any other version goes to `failed/` |
| `source.app`, `source.version`, `source.exported` | no | origin, display only |
| `created` | no | ISO 8601 |
| `playlist.kind` | no | `rb_playlist` or `list`. Missing = `list` |
| `playlist.id`, `playlist.name` | no | the name is used for the tidal playlist |
| `tracks[]` | yes | at least one readable title |
| `tracks[].position` | no | order, default 0 |
| `tracks[].id` | no | key for the match cache. Missing: `source.bandcamp.trackId`, else `pageUrl#trackIndex` |
| `tracks[].artist`, `tracks[].title` | yes | a track without them is skipped and counted |
| `tracks[].version`, `album`, `label`, `isrc`, `localFile` | no | `null` or missing = empty |
| `tracks[].duration` | no | seconds, 0 = unknown |
| `tracks[].tidalId` | no | taken over, not searched again |

Rules:

- unknown fields are ignored, so new fields may be added.
- a title without artist or title is skipped; the file stays valid.
- a file without any readable title, or with another version, goes to `failed/`.
- Charon matches each title on tidal (never during a recording), and classifies it as found, unsure (title equal, duration differs or title partly equal), or not on tidal.
- matches are cached by `tracks[].id`. Confirmed decisions survive a new search.
- creating the tidal playlist happens only after a dry-run preview. The new playlist is not public. Existing playlists are never changed.
- Spotify export: not yet.
