# Tony's Salesforce Skills

Public Salesforce AI skills for use with Claude Code and compatible skill runners.

## Skills

### Test Generation

| Skill | Description |
|---|---|
| `generating-apex-test-httpmocks` | Generate Apex test classes for HTTP callout code using the self-shunt pattern |

### Clean Code for Apex

Adapted from *Clean Code* (Robert C. Martin) — authoring skills applied as you write, `reviewing-apex` for existing code.

| Skill | Source | Use when |
|---|---|---|
| `clean-apex-naming` | Ch 2 — Meaningful Names | Writing or renaming variables, methods, classes, constants |
| `clean-apex-functions` | Ch 3 — Functions | Writing or refactoring methods, handlers, service logic |
| `clean-apex-comments` | Ch 4 — Comments | Writing or cleaning up comments; removing dead code |
| `clean-apex-error-handling` | Ch 7 — Error Handling | Callouts, DML services, exceptions, null handling |
| `reviewing-apex` | Ch 17 + 6, 9, 10 | Reviewing or critiquing existing Apex code |

## Installation

```bash
npx skills add tonygward/salesforce-skills --yes
```

## Usage

Once installed, invoke a skill by name in your Claude Code session:

```
/generating-apex-test-httpmocks
/clean-apex-naming
/clean-apex-functions
/clean-apex-comments
/clean-apex-error-handling
/reviewing-apex
```

## Contributing

Each skill lives in `skills/<name>/SKILL.md`. PRs welcome.
