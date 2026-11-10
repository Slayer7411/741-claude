# Release hook

Each 741 app's release script runs `tools/build-skill.sh` from this repo (with `DEV_ROOT` pointing at the folder that holds the source repos), then commits and pushes this repo. The script regenerates `plugins/741/skills/741/references/` from the current docs, passes everything through the name and personal-detail sanitizer (it fails on a leftover name), bumps the date-based plugin version in `plugin.json` and `marketplace.json`, and runs `claude plugin validate .`.
