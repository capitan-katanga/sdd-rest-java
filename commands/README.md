# commands — sdd-rest-java

## What lives here

Slash commands that the user types into Claude Code to trigger predefined flows. All filenames follow the convention `sddrest.<verb>.md`; the user invokes them as `/sdd-rest-java:sddrest.<verb>`.

Current roster (11):

| Command | Purpose |
|---|---|
| `sddrest.architect-review` | High-level architecture review of the current branch |
| `sddrest.code-review` | PR-style code review |
| `sddrest.dependency-audit` | Audit Maven deps for CVEs and outdated versions |
| `sddrest.generate-crud` | Scaffold full CRUD (entity, repo, service, controller, tests) for a domain class |
| `sddrest.generate-docs` | Generate Javadoc / OpenAPI / README updates |
| `sddrest.generate-refactoring-tasks` | Produce a refactoring task list for a target class or package |
| `sddrest.refactor-class` | Apply a refactor to a specific class |
| `sddrest.security-review` | Security-focused review (JWT, secrets, injection, OWASP) |
| `sddrest.upgrade-dependencies` | Coordinated dependency upgrade plan |
| `sddrest.write-integration-tests` | Generate Testcontainers / WireMock integration tests |
| `sddrest.write-unit-tests` | Generate unit tests for a target file |

## Loading model

Loaded **on demand** when the user types the slash command. The body of the `.md` file is passed to Claude as the prompt for that turn. Commands typically delegate to one or more agents and/or skills.

Plugin-namespaced form (always works): `/sdd-rest-java:sddrest.<verb>`. If no other plugin exposes the same `sddrest.*` namespace, the short form `/sddrest.<verb>` also works.

## File structure

```markdown
---
description: >
  What this command does and when to use it. Surfaced in /help and command pickers.
argument-hint: "[domain-class-name]"   # placeholder shown to the user
allowed-tools: Read, Write, Bash       # tool allowlist for this command
model: inherit                         # opus | sonnet | haiku | inherit
---

# <Command title>

<instructions Claude follows when this command runs — what to do, which agents/skills to invoke, what output to produce>
```

| Frontmatter field | Required | Notes |
|---|---|---|
| `description` | yes | Single sentence; should make the use case obvious. |
| `argument-hint` | when arguments are expected | Format: bracketed placeholder, e.g. `"[domain-class-name]"`. |
| `allowed-tools` | yes | Comma-separated list of tools the command may use. |
| `model` | recommended | `inherit` keeps the user's current model; otherwise pick explicitly. |

## How to add a command

1. Create `commands/sddrest.<verb>.md` with the frontmatter above. Use a clear verb (`generate-crud`, `security-review`).
2. Write the body so it tells Claude exactly what to do — the typical pattern is: load context, invoke a skill or agent, produce a structured output.
3. Register it in `.claude-plugin/plugin.json` under `commands`:
   ```json
   "commands": [
     "...existing...",
     "./commands/sddrest.<verb>.md"
   ]
   ```
4. Restart Claude Code.
5. Smoke test: type `/sdd-rest-java:sddrest.<verb>` and confirm Claude follows the script.

## How to remove a command

1. Delete `commands/sddrest.<verb>.md`.
2. Remove the entry from the `plugin.json` `commands` array.
3. Find stale references:
   ```bash
   grep -rln 'sddrest\.<verb>' /path/to/sdd-rest-java/
   ```
   Likely hits: README, other commands that chain into it.
4. Restart Claude Code.

## Authoring conventions

- Keep the body short: the command is a script, not a tutorial. Heavy domain knowledge belongs in skills, not commands.
- Prefer **delegating to agents** for multi-step work. A command of more than ~80 lines is usually doing too much.
- If the command takes input, validate it in the first paragraph and fail loudly on missing arguments.
- Don't repeat skill content. Reference the skill by name; Claude will load it when triggers match.

## Validation

```bash
PLUGIN=/absolute/path/to/sdd-rest-java

python3 -c "
import json, os
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
for p in m['commands']:
    assert os.path.exists(f'$PLUGIN/{p[2:]}'), p
print('OK')
"

# Frontmatter present in each command
for f in $PLUGIN/commands/sddrest.*.md; do
  head -1 "$f" | grep -q '^---$' || echo "BAD frontmatter: $f"
done

# Filenames follow the sddrest.* convention
for f in $PLUGIN/commands/*.md; do
  case "$(basename "$f")" in
    README.md|sddrest.*.md) ;;
    *) echo "naming violation: $f" ;;
  esac
done
```

## See also

- [../agents/README.md](../agents/README.md) — agents that commands typically delegate to
- [../skills/README.md](../skills/README.md) — skills that commands rely on
- [../.claude-plugin/README.md](../.claude-plugin/README.md) — manifest format
