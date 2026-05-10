# rules — sdd-rest-java

## What lives here

Always-on guidance that Claude applies to every conversation in a project that has the plugin installed. Rules are coding standards, naming conventions, and project-structure expectations — the things that should never need to be re-explained per task.

Current rules (4):

| File | Purpose |
|---|---|
| `error-handling.md` | Exception hierarchy, RFC 7807 Problem Details, recovery patterns |
| `language-best-practices.md` | Modern Java idioms (records, sealed types, pattern matching) |
| `naming-conventions.md` | Package, class, method, and field naming standards |
| `project-structure.md` | Maven layout, package-by-feature vs by-layer, module boundaries |

## Loading model

Rules are **always loaded** in every conversation when the plugin is active. They are *not* listed in `plugin.json` — Claude Code picks them up by directory convention (`rules/*.md`).

The `paths:` glob in the frontmatter narrows which files the rule applies to. A rule with `paths: ["**/*.java"]` only constrains Claude when it touches Java files, even though the rule itself is always in context.

## File structure

```markdown
---
paths:
  - "**/*.java"           # globs that scope where this rule applies
---
# Rule: <title>

## Context
<one paragraph: what this rule exists to enforce>

## Guidelines
- <concrete, testable rule>
- <another concrete rule>

## Examples
<good vs. bad code>

## Rationale
<short justification — helps Claude judge edge cases>
```

| Frontmatter field | Required | Notes |
|---|---|---|
| `paths` | yes | Array of glob patterns. Use `["**/*"]` only if the rule truly applies everywhere. |

## How to add a rule

1. Create `rules/<topic>.md` with the frontmatter above.
2. Write rules as concrete, testable bullet points — *"prefer `Optional<T>` over null returns"*, not *"write good code"*.
3. Always include a **Rationale** so Claude can apply the rule to cases the bullets didn't anticipate.
4. No `plugin.json` change needed — the directory is scanned automatically.
5. Restart Claude Code.

## How to remove a rule

1. Delete `rules/<topic>.md`.
2. `grep -rln '<rule-filename>' /path/to/sdd-rest-java/` to find references in other docs (rare — rules don't usually cross-reference each other).
3. Restart Claude Code.

## Authoring conventions

- **One topic per file.** Don't bundle naming + error handling.
- **Be enforceable.** A rule should be something Claude can verify against the code in front of it.
- **Scope tightly with `paths`.** A Java rule has no business firing on `*.tf` or `*.yml`.
- **Avoid duplicating skills.** Skills are *task-specific* knowledge loaded by triggers; rules are *project-wide* invariants. If you find yourself writing "when implementing CRUD…", that belongs in a skill.

## Validation

```bash
PLUGIN=/absolute/path/to/sdd-rest-java

# Every rule has frontmatter with a paths field
for f in $PLUGIN/rules/*.md; do
  [ "$(basename "$f")" = "README.md" ] && continue
  head -5 "$f" | grep -q '^paths:' || echo "missing paths: $f"
done
```

## See also

- [../skills/README.md](../skills/README.md) — task-scoped, trigger-loaded knowledge (the inverse of rules)
- [../README.md](../README.md)
