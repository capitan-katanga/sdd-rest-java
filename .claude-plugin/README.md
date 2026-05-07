# .claude-plugin — sdd-rest-java

## What lives here

`plugin.json` — the plugin manifest. It is the single source of truth for which skills, agents, and commands ship with this plugin. Claude Code reads this file when the plugin is loaded; anything not listed here is invisible to Claude even if it sits on disk.

## Loading model

Loaded **once at plugin startup**. Editing `plugin.json` requires restarting Claude Code to take effect.

## Schema

```json
{
  "name": "sdd-rest-java",
  "version": "0.1.0",
  "description": "...",
  "author":   { "name": "...", "email": "..." },
  "license":  "MIT",
  "keywords": ["java", "spring-boot-4", ...],
  "agents":   ["./agents/<name>.md", ...],
  "commands": ["./commands/sddrest.<name>.md", ...],
  "skills":   ["./skills/<name>", ...]
}
```

| Field | Purpose |
|---|---|
| `name` | Plugin identifier. Used in slash command namespace (`/sdd-rest-java:sddrest.<cmd>`). Must match the directory name. |
| `version` | SemVer string. Bump when changing the plugin's behavior in a way consumers should notice. |
| `description` | Short summary; surfaced in marketplaces and listings. |
| `agents` | Array of paths (relative, with `./` prefix) to agent `.md` files. |
| `commands` | Array of paths to slash command `.md` files. |
| `skills` | Array of paths to **skill directories** (not files — Claude looks for `<dir>/SKILL.md` inside each). |

## How to add an artifact

1. Create the file (or directory, for skills) on disk under `agents/`, `commands/`, or `skills/`.
2. Add the relative path to the matching array in `plugin.json`. Order is preserved but does not affect routing.
3. Restart Claude Code.

See the per-directory READMEs for the structure of each artifact:
- [../agents/README.md](../agents/README.md)
- [../commands/README.md](../commands/README.md)
- [../rules/README.md](../rules/README.md) — note: rules are not listed in `plugin.json`; they live in `rules/` and are loaded by directory convention
- [../skills/README.md](../skills/README.md)

## How to remove an artifact

1. Remove the entry from the matching array in `plugin.json`.
2. Delete the file or directory on disk.
3. `grep -rln '<removed-name>' ..` from this directory to find stale references in agents, commands, or skill bodies. Replace or delete them.
4. Restart Claude Code.

## When to bump `version`

- **PATCH** (`0.1.0` → `0.1.1`): typo fixes, clarifications inside skill/agent bodies, no behavioral change.
- **MINOR** (`0.1.0` → `0.2.0`): new skill, new command, new agent, or expanded triggers.
- **MAJOR** (`0.1.0` → `1.0.0`): renaming or removing a skill/agent/command, or a breaking change to a frontmatter contract.

## Validation

Run from the plugin root after any edit:

```bash
PLUGIN=/absolute/path/to/sdd-rest-java

# 1. Every path in plugin.json exists on disk
python3 -c "
import json, os
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
for p in m['skills'] + m['agents'] + m['commands']:
    assert os.path.exists(f'$PLUGIN/{p[2:]}'), p
print('OK')
"

# 2. Skill count matches the number of directories on disk
test $(find $PLUGIN/skills -mindepth 1 -maxdepth 1 -type d | wc -l) \
  -eq $(python3 -c "import json; print(len(json.load(open('$PLUGIN/.claude-plugin/plugin.json'))['skills']))") \
  && echo "skill count OK"
```

Both should print `OK` / `skill count OK`. Any other output means the manifest and disk are out of sync.

## See also

- [../INSTALL.md](../INSTALL.md)
- [../README.md](../README.md)
