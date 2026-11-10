# ◆ HELIOS

installer, updater and setup of the 741 environment. Planned, not built. The repository holds the plan only. Every app keeps working without HELIOS.

## ◆ STATUS

- planned. No shipped features yet.
- the plan's first step is the release feed and catalog for each app. Then a command-line core (catalog, feed check, download, verify, install, rollback), then the setup flow, shared settings, and a window.
- until HELIOS ships: each app has its own updater and setup. Stand-alone mode works without the agent.

## ◆ WHAT IT WILL DO

- create or adopt the 741 home (Drive folder or local folder), write `agent.json`, install and start `mnemo-agent` as a LaunchAgent, and check the hello reply
- hold shared settings (music library folders, streaming folder, export, backups) as `setting` records
- install, update, roll back and remove MINOS, ORPHEUS, CHARON, STYX and the agent
- install the 741 skill as a Claude plugin (optional, off by default)

