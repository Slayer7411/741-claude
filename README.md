# ◆ 741-CLAUDE

Claude Code plugin marketplace for the 741 tools. It ships one skill, `741`, which gives Claude:

- what the `.741` collection file is and how the apps share it
- how to change a collection safely (through `mnemo-agent`, with backups)
- what each 741 app does: MINOS, ORPHEUS, CHARON, STYX, HELIOS

## ◆ INSTALL

```
claude plugin marketplace add OWNER/741-claude
claude plugin install 741@741-claude
```

## ◆ LAYOUT

- `.claude-plugin/marketplace.json` · marketplace with one plugin
- `plugins/741/` · the plugin
- `plugins/741/skills/741/SKILL.md` · entry point: what 741 is, hard rules, which reference to load
- `plugins/741/skills/741/references/` · format, writing, one file per app (generated)
- `tools/parts/` · hand-written parts of the references
- `tools/build-skill.sh` · regenerates the references from the source repos, runs the name check, bumps the version, runs `claude plugin validate .`

## ◆ BUILD

```
DEV_ROOT=<folder that holds the source repos> tools/build-skill.sh
```

`DEV_ROOT` defaults to the parent folder of this repo. Needs the `claude` CLI.

## ◆ STATUS

The format is a draft (schema 1, not released). Planned features are marked as planned in the references.
