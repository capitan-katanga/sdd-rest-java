# Installing sdd-rest-java

This plugin is a Claude Code plugin. It runs inside Claude Code (CLI, IDE, or web app) — there is no separate runtime, no daemon, and nothing to deploy.

## Prerequisites

- Claude Code installed (`claude` CLI, IDE extension, or web app)
- A target Spring Boot 4.x project (Java 25, Jakarta EE 11, Spring Framework 7)
- The plugin source — either cloned locally or available through a marketplace

## Option A — Local install (recommended while iterating)

Point Claude Code at the plugin directory on disk by adding it to a `settings.json`.

**User-level** (`~/.claude/settings.json` — applies everywhere):

```json
{
  "plugins": [
    "/home/ignacioencizo/aiprojects/java-backend-agent-project/sdd-rest-java"
  ]
}
```

**Project-level** (`.claude/settings.json` at the root of a specific Spring Boot project — applies only there):

```json
{
  "plugins": [
    "/absolute/path/to/sdd-rest-java"
  ]
}
```

Project-level wins over user-level for that project. Use project-level when only one repo should consume the plugin; use user-level when you want it active in every Spring Boot project on your machine.

The path must be absolute. Restart Claude Code after editing the file.

## Option B — Marketplace install

If the plugin has been published to a marketplace, reference it by name instead of by path:

```json
{
  "plugins": ["sdd-rest-java"]
}
```

Use this once you stop iterating on the plugin source.

## Verifying the install

Open Claude Code in any directory and ask:

```
list available skills, agents, and slash commands from sdd-rest-java
```

You should see:
- 41 skills (including `spring-boot-rest-api-standards`, `spring-mvc-testing`, `core-setup`, …)
- 7 agents (`spring-boot-backend-development-expert`, `spring-boot-code-review-expert`, …)
- 11 slash commands prefixed `sddrest.` (`sddrest.generate-crud`, `sddrest.security-review`, …)

Or run a structural validation against the plugin source on disk:

```bash
PLUGIN=/absolute/path/to/sdd-rest-java
python3 -c "
import json, os
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
for p in m['skills'] + m['agents'] + m['commands']:
    assert os.path.exists(f'$PLUGIN/{p[2:]}'), p
print('plugin.json OK:', len(m['skills']), 'skills,', len(m['agents']), 'agents,', len(m['commands']), 'commands')
"
```

Expected output: `plugin.json OK: 41 skills, 7 agents, 11 commands`.

## Post-install smoke test

Inside an actual Spring Boot 4.x project, ask Claude:

```
create a REST /orders endpoint with CRUD, JPA persistence, and unit + slice tests
```

Confirm in Claude's response that it loaded the relevant skills (e.g., `spring-boot-rest-api-standards`, `spring-boot-crud-patterns`, `spring-data-jpa`, `spring-mvc-testing`, `spring-jpa-testing`). If those triggers do not fire, the plugin is not installed correctly — re-check the path in `settings.json`.

## Uninstalling

Remove the plugin entry from `settings.json` and restart Claude Code. The plugin source on disk is untouched.

## See also

- [README.md](README.md) — what the plugin contains, how to use it day-to-day
- [.claude-plugin/README.md](.claude-plugin/README.md) — the manifest (`plugin.json`)
- [skills/README.md](skills/README.md) — adding, removing, and authoring skills
