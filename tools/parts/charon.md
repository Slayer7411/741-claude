# ◆ CHARON

native tidal recorder for macos. Captures only the output of the tidal player process, bit-exact, with sample-exact track cuts, a verify verdict, and tags and cover from tidal. Private, for a personal archive. The sections below are the app's README and its handover format.

## ◆ 741 STATUS

- the README (section 741) describes Charon registering each finished recording in `collection.741` through the agent. Per track: tags, ISRC, tidal id in `streams`, technical data, file path under the music root, verdict in `x.charon.verdict`. Per queue run: one playlist (default name `Charon {date}`).
- duplicate check before recording: by tidal id or ISRC. Policy `ask` (default), `skip` or `record`.
- without an agent, writes wait in the client outbox. Recording never waits for the agent.
- settings are in `settings.json` today (the `coll741*` keys). Target: `setting` records of app `charon`.
- the integration plan lists the Charon reads of the nml collection and the handover file as removed. The README is the current description of what the app does.

## ◆ HANDOVER FROM STYX

STYX places a playlist file in the Charon inbox. Format and rules: section below.

