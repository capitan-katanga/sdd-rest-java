# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`sdd-rest-java` is a **Claude Code plugin** — static markdown + a JSON manifest. It is *not* a runtime Java application. There is no build, no compile, no test runner; nothing executes here. The plugin ships knowledge (skills, agents, commands, rules) that Claude Code loads when the plugin is installed in a target Spring Boot 4.x project.

Implication: edits to this repo are content edits to markdown/JSON. Verify them with the validation snippets below, not with `mvn`/`gradle`.

## The four artifact types and how they load

These differ in *when* they enter Claude's context. Confusing the loading model is the most common authoring mistake.

| Type | Lives in | Listed in `plugin.json`? | Loads when |
|---|---|---|---|
| **Skills** (35) | `skills/<name>/SKILL.md` | yes | Frontmatter `description` matches the conversation. Body loads only on match. `references/*.md` load only if SKILL.md cites them by path. |
| **Agents** (7) | `agents/<name>.md` | yes | Invoked by name (user request, slash command, or another agent). Runs in a fresh subagent context. |
| **Commands** (10) | `commands/sddrest.<verb>.md` | yes | User types `/sdd-rest-java:sddrest.<verb>`. The `.md` body becomes the prompt. |
| **Rules** (4) | `rules/<topic>.md` | **no** — directory convention | Always-on in every conversation. `paths:` glob narrows where the rule applies, but the rule is always in context. |

`plugin.json` is the source of truth for skills/agents/commands — anything not listed there is invisible to Claude even if the file exists on disk. Rules are picked up by scanning `rules/`.

**Restart Claude Code after editing `plugin.json` or adding/removing artifact files.** The manifest is read once at startup.

## Skill authoring — the load-bearing detail

For skills, Claude reads only the frontmatter `description` to decide whether to load the body. The description *is* the routing filter. Two consequences:

- Triggers must be specific: class names, annotations (`@WebMvcTest`), file extensions, config property paths. Vague triggers (`"Java"`, `"backend"`, `"API"`) cause false positives that inflate context cost on every conversation.
- When two skills overlap (e.g. `spring-mvc-testing` vs `spring-security-testing`), use a **`Do NOT Use This Skill When`** section to hand off territory explicitly. This is the cheapest fix for trigger collisions.
- `references/*.md` are *not* auto-loaded. They only enter context when SKILL.md cites them by path. An uncited reference is dead weight.

Bump the SKILL.md frontmatter `version` when you make a meaningful change, so consumers can track behavior shifts.

## Naming conventions (enforced)

- **Skill directory name == frontmatter `name`.** Same string in both places.
- **Agent filename (without `.md`) == frontmatter `name`.**
- **Commands** must be `commands/sddrest.<verb>.md`. The `sddrest.` prefix is the namespace — anything else is a naming violation flagged by the validation script.
- Plugin manifest entries use `./` prefix and are relative paths (e.g. `"./skills/<name>"`, `"./agents/<name>.md"`).

## Add / remove workflow

Adding any artifact (except a rule) is two steps: create the file/directory, then register the path in `.claude-plugin/plugin.json`. Removing requires three: delete the file, remove the manifest entry, and `grep -rln '<removed-name>' .` to find stale references in other agents' `skills:` lists, command bodies, READMEs, etc. Stale references make Claude recommend things that no longer exist.

For rules: no manifest edit — just add/remove the file in `rules/`.

The per-directory READMEs (`agents/README.md`, `commands/README.md`, `rules/README.md`, `skills/README.md`, `.claude-plugin/README.md`) document the frontmatter contract for each artifact type. Consult them when authoring.

## Validation

Run from the plugin root after any structural edit. Both should print `OK` (and `skill count OK`):

```bash
PLUGIN=/home/ignacioencizo/aiprojects/sdd-rest-java

# 1. Every path in plugin.json resolves on disk; every skill has a SKILL.md
python3 -c "
import json, os
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
for p in m['skills']:
    assert os.path.isdir(f'$PLUGIN/{p[2:]}'), p
    assert os.path.exists(f'$PLUGIN/{p[2:]}/SKILL.md'), p + '/SKILL.md'
for p in m['agents'] + m['commands']:
    assert os.path.exists(f'$PLUGIN/{p[2:]}'), p
print('OK')
"

# 2. Skill directory count matches the manifest
test $(find $PLUGIN/skills -mindepth 1 -maxdepth 1 -type d | wc -l) \
  -eq $(python3 -c "import json; print(len(json.load(open('$PLUGIN/.claude-plugin/plugin.json'))['skills']))") \
  && echo "skill count OK"

# 3. Frontmatter present on every artifact
for f in $PLUGIN/agents/*.md $PLUGIN/commands/sddrest.*.md $PLUGIN/rules/*.md; do
  [ "$(basename "$f")" = "README.md" ] && continue
  head -1 "$f" | grep -q '^---$' || echo "BAD frontmatter: $f"
done
for s in $PLUGIN/skills/*/; do
  head -1 "$s/SKILL.md" | grep -q '^---$' || echo "BAD frontmatter: $s"
done
```

Expected baseline counts: **29 skills, 7 agents, 10 commands, 4 rules**.

 Mismatches mean the manifest and disk are out of sync.

## Versioning

`plugin.json` `version` follows SemVer: PATCH for typos/clarifications, MINOR for new skills/commands/agents or expanded triggers, MAJOR for renames, removals, or breaking frontmatter changes.

The plugin tracks the **Spring Boot 4.x line** (Java 25, Jakarta EE 11, Spring Framework 7). Boot 3.x is *not* a target — references to legacy patterns belong only in migration sections of relevant skills.

## Out of scope

AI/ML, LangChain4j, Spring AI MCP, vector stores. Don't add skills/agents/commands in those areas without an explicit scope change.
