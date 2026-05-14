# agents — sdd-rest-java

## What lives here

Subagents specialized for Java/Spring Boot backend work. Each `.md` file defines one agent with its own model, tool allowlist, and a curated list of skills it knows about.

Current roster (7):

| Agent | Purpose |
|---|---|
| `spring-boot-backend-development-expert` | Feature implementation, REST endpoints, architecture decisions |
| `spring-boot-code-review-expert` | PR-style review of Spring Boot code |
| `spring-boot-unit-testing-expert` | Authoring and fixing unit / slice tests |
| `java-refactor-expert` | Targeted refactors — extract, rename, restructure |
| `java-security-expert` | Security audits, JWT/OAuth review, secret handling |
| `java-software-architect-review` | Higher-level architectural review |
| `java-documentation-specialist` | In-code documentation under the layer split — OpenAPI annotations on `@RestController`, JEP 467 `///` Markdown Javadoc on services/repositories/exceptions |

## Loading model

Agents are **invoked by name**, not loaded automatically. Three ways they get used:

1. The user asks for one explicitly: *"use the `spring-boot-code-review-expert` to review this PR"*.
2. A slash command in `commands/` invokes one (e.g. `sddrest.code-review` calls `spring-boot-code-review-expert`).
3. Another agent or skill recommends one.

When invoked, the entire agent body is loaded into a **fresh subagent context**. The parent conversation does not see the subagent's intermediate work — only its final report.

## File structure

```markdown
---
name: <agent-name>             # must match the filename without .md
description: >                 # how Claude decides when to suggest this agent
  Provides expert ... Use proactively when ...
tools: [Read, Write, Edit, Glob, Grep, Bash]   # tool allowlist for this agent
model: sonnet                  # opus | sonnet | haiku | inherit
skills:                        # skills this agent should be primed on
  - spring-boot-crud-patterns
  - spring-boot-dependency-injection
---

# <Agent display name>

<role, instructions, output format, …>
```

| Frontmatter field | Required | Notes |
|---|---|---|
| `name` | yes | Slug; must match filename. |
| `description` | yes | Trigger text for routing. Be specific about *when* to use this agent vs. another. |
| `tools` | yes | Subset of available tools. Omit destructive tools the agent shouldn't have. |
| `model` | recommended | Pick the cheapest model that handles the task. Reviews/architecture tend to need sonnet or opus; doc-only agents can use haiku. |
| `skills` | optional | Skills to surface to the agent. The agent still loads other skills if their triggers match. |

## How to add an agent

1. Create `agents/<agent-name>.md` with the frontmatter contract above.
2. Write the agent body. Keep it focused — one role per agent.
3. Register it in `.claude-plugin/plugin.json` under `agents`:
   ```json
   "agents": [
     "...existing...",
     "./agents/<agent-name>.md"
   ]
   ```
4. If a slash command should drive this agent, update or create the corresponding command in `commands/`.
5. Restart Claude Code.

## How to remove an agent

1. Delete `agents/<agent-name>.md`.
2. Remove the entry from `plugin.json` `agents` array.
3. Find stale references and decide what to do with each:
   ```bash
   grep -rln '<agent-name>' /path/to/sdd-rest-java/
   ```
   Look in `commands/` (slash commands that invoked it) and other agents (`skills:` lists that mention it).
4. Replace references with a substitute agent or remove them. Stale references mean Claude may recommend an agent that no longer exists.
5. Restart Claude Code.

## Validation

```bash
PLUGIN=/absolute/path/to/sdd-rest-java

# Every registered agent file exists
python3 -c "
import json, os
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
for p in m['agents']:
    assert os.path.exists(f'$PLUGIN/{p[2:]}'), p
print('OK')
"

# Every agent file has frontmatter
for f in $PLUGIN/agents/*.md; do
  [ "$(basename "$f")" = "README.md" ] && continue
  head -1 "$f" | grep -q '^---$' || echo "BAD frontmatter: $f"
done
```

## See also

- [../skills/README.md](../skills/README.md) — what the `skills:` frontmatter list points to
- [../commands/README.md](../commands/README.md) — how slash commands invoke agents
- [../.claude-plugin/README.md](../.claude-plugin/README.md) — manifest format
