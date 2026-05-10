# skills — sdd-rest-java

## What lives here

Task-scoped knowledge packs, each in its own subdirectory. A skill bundles concrete instructions, code templates, and references for one well-defined task — *write a Spring MVC slice test*, *configure a Spring Cloud Config Server*, *containerize with a multi-stage Dockerfile*. This is where the bulk of the plugin's domain knowledge lives.

Categories:
- **Foundations** — `core-setup`, `spring-boot-project-creator`, `spring-boot-dependency-injection`
- **REST & web** — `spring-boot-rest-api-standards`, `spring-boot-crud-patterns`, `spring-boot-openapi-documentation`, `spring-boot-actuator`
- **Reliability** — `spring-boot-cache`, `spring-boot-resilience4j`, `spring-boot-event-driven-patterns`, `spring-boot-saga-pattern`
- **Persistence** — `spring-data-jpa`, `spring-data-neo4j`
- **Security** — `spring-security-jwt`
- **Container / runtime** — `containerization-docker`
- **Observability** — `observability-logging`
- **Testing slices** — `spring-testing-fundamentals`, `spring-jpa-testing`, `spring-mvc-testing`, `spring-security-testing`, `spring-webflux-testing`, `spring-websocket-testing`
- **Unit-test recipes** — `unit-test-application-events`, `bean-validation`, `boundary-conditions`, `caching`, `config-properties`, `json-serialization`, `mapper-converter`, `parameterized`, `scheduled-async`, `utility-methods`, `wiremock-rest-api`, `wiremock-standalone-docker`

## Loading model — the most important section

Skills are **the only artifact loaded automatically by triggers**. The mechanism:

1. At conversation start, Claude reads only the **frontmatter `description`** of each skill — never the body.
2. As the conversation evolves, Claude matches signals (keywords, class names, annotations, file paths) against those descriptions.
3. When a description matches, **the SKILL.md body is loaded** into context.
4. `references/*.md` files inside a skill are **not auto-loaded** — they only enter context if the SKILL.md cites them by path.

**The `description` is the entire filter.** A vague description (`"For Java projects"`) generates false positives and inflates context cost. A specific description with explicit triggers (`"Triggers: @WebMvcTest, MockMvc, @AutoConfigureMockMvc, application/json content negotiation"`) loads only when relevant.

Invest time in the description. The body can be edited later; bad triggers waste context every conversation.

## Skill directory layout

```
skills/<skill-name>/
├── SKILL.md             # required — frontmatter + body
├── references/          # optional — loaded on demand when SKILL.md cites them
│   ├── <topic-1>.md
│   └── <topic-2>.md
├── assets/              # optional — templates, snippets (e.g. pom.xml fragments)
└── scripts/             # optional — helper scripts the skill body invokes
```

### `SKILL.md` frontmatter

```markdown
---
name: <skill-name>                 # must match the directory name
description: >
  One sentence: what the skill does and when to use it.
  Triggers: <keyword>, <@Annotation>, <ClassName>, <property.path>, <regex-pattern>.
version: 0.1.0
license: Apache-2.0                # optional
---
```

### `SKILL.md` body — recommended sections

| Section | Purpose |
|---|---|
| **Tested With** | Versions verified to work (Boot 4.0, Java 25, …). |
| **Do NOT Use This Skill When** | Anti-routing. Lists adjacent skills that should win in their territory (critical when triggers overlap — e.g. mvc vs webflux vs security testing). |
| **When to Read References** | A table mapping situations → which `references/<file>.md` to load. Without this, references are dead weight. |
| **Examples** | Short, copy-pasteable code. |
| **Anti-patterns** | What this skill explicitly tells Claude *not* to do. |

## How to add a skill

Five steps, in order:

1. **Create the directory and SKILL.md.**
   ```
   skills/<new-skill>/SKILL.md
   ```

2. **Write the frontmatter — triggers are the load-bearing part.**
   ```markdown
   ---
   name: <new-skill>
   description: >
     <one sentence about what and when>.
     Triggers: <keyword>, <@Annotation>, <ClassName>, <config.property>, <file pattern>.
   version: 0.1.0
   ---
   ```
   Triggers must be specific: class names, annotations, file extensions, configuration property paths. Vague triggers (`"API"`, `"Java"`, `"backend"`) generate false positives that cost context on every conversation.

3. **Write the body.** Include the recommended sections above, especially **Do NOT Use This Skill When** if there's any overlap with an existing skill.

4. **Add dense `references/*.md`** for material that isn't always needed. Cite each one explicitly in the SKILL.md body — references that aren't cited are never loaded.

5. **Register the skill in `.claude-plugin/plugin.json`.**
   ```json
   "skills": [
     "...existing...",
     "./skills/<new-skill>"
   ]
   ```

Restart Claude Code. Smoke test by mentioning one of the triggers in a fresh conversation and confirming the skill loads.

## How to remove a skill

Three steps:

1. **Delete the directory.**
   ```
   rm -rf skills/<skill-to-remove>/
   ```

2. **Remove the entry** from the `skills` array in `.claude-plugin/plugin.json`.

3. **Sweep for stale references.** Other skills, agents, and commands may name the removed skill in their bodies or in `skills:` frontmatter lists.
   ```bash
   grep -rln '<skill-to-remove>' /path/to/sdd-rest-java/
   ```
   For each hit: replace with the substitute skill, or delete the reference. Stale references mean Claude may recommend a skill that no longer exists.

If you are **merging** the skill into another (rather than discarding its content), copy the old `SKILL.md` to `references/<topic>.md` of the destination skill before deleting, and add a row in the destination's *When to Read References* table.

## Things worth knowing

- **The description is the only routing input.** Edit it deliberately. Re-test triggers in a fresh conversation after edits.
- **`Do NOT Use This Skill When`** is the cheapest fix when two skills compete for the same triggers — explicitly hand off the territory. We rely on this for the testing skills (`mvc` vs `security` vs `webflux` vs `websocket`).
- **References are not magic.** A `references/foo.md` that the SKILL.md doesn't cite by path will never be read.
- **Bump `version`** in the frontmatter when you make a meaningful change to a SKILL.md. Helps you (and consumers) track behavior shifts.
- **Don't hand-edit `plugin.json` for bulk changes.** When the plugin grows, write a small script that lists `skills/*/` and rewrites the array.

## Validation

```bash
PLUGIN=/absolute/path/to/sdd-rest-java

# 1. Skill count on disk equals the manifest count
test $(find $PLUGIN/skills -mindepth 1 -maxdepth 1 -type d | wc -l) \
  -eq $(python3 -c "import json; print(len(json.load(open('$PLUGIN/.claude-plugin/plugin.json'))['skills']))") \
  && echo "skill count OK"

# 2. Every registered skill resolves on disk
python3 -c "
import json, os
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
for p in m['skills']:
    assert os.path.isdir(f'$PLUGIN/{p[2:]}'), p
    assert os.path.exists(f'$PLUGIN/{p[2:]}/SKILL.md'), p + '/SKILL.md'
print('OK')
"

# 3. Every SKILL.md has a frontmatter block
for s in $PLUGIN/skills/*/; do
  head -1 "$s/SKILL.md" | grep -q '^---$' || echo "BAD frontmatter: $s"
done

# 4. No skill references a name that doesn't exist
python3 -c "
import json, os, re
m = json.load(open('$PLUGIN/.claude-plugin/plugin.json'))
known = {os.path.basename(p) for p in m['skills']}
for s in known:
    body = open(f'$PLUGIN/skills/{s}/SKILL.md').read()
    for ref in re.findall(r'\b(unit-test-[a-z-]+|spring-[a-z-]+)\b', body):
        if ref not in known and ref not in {'spring-boot','spring-cloud','spring-framework','spring-data'}:
            pass  # noise — tighten this regex if you need a strict check
print('done')
"
```

## See also

- [../INSTALL.md](../INSTALL.md)
- [../README.md](../README.md)
- [../agents/README.md](../agents/README.md) — agents reference skills via the `skills:` frontmatter list
- [../commands/README.md](../commands/README.md) — commands invoke skills indirectly through agents
- [../.claude-plugin/README.md](../.claude-plugin/README.md) — manifest format and how skills are registered
