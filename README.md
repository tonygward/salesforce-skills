# Tony's Salesforce Skills

Public Salesforce AI skills for use with Claude Code and compatible skill runners.

## Installation

```bash
npx skills add tonygward/salesforce-skills --yes
```

## Usage

Once installed, invoke a skill by name in your Claude Code session:

```
/salesforce-naming-things
/salesforce-code-comments
/salesforce-writing-tests
/salesforce-tests-as-specification
/salesforce-tdd-workflow
/generating-apex-test-httpmocks
/clean-apex-functions
/clean-apex-error-handling
/reviewing-apex
```

## Skills

### Salesforce Fundamentals

| Skill | Description |
|---|---|
| `salesforce-naming-things` | Naming conventions for Apex and LWC — intent-revealing names, no encodings, Salesforce conventions |
| `salesforce-code-comments` | When and how to comment Salesforce code — the why-not-how rule, ApexDoc/JSDoc, what to delete |

### Testing

| Skill | Description |
|---|---|
| `salesforce-writing-tests` | Write high-quality Apex and LWC Jest tests — FIRST principles, Assert class, bulk testing, mocking |
| `salesforce-tests-as-specification` | Tests as a living spec — one rule per test, Given/When/Then, boundary cases |
| `salesforce-tdd-workflow` | TDD workflow for Salesforce — red/green/refactor, three laws of TDD, one change at a time |
| `generating-apex-test-httpmocks` | Generate Apex test classes for HTTP callout code using the self-shunt pattern |

### Clean Code for Apex

Adapted from *Clean Code* (Robert C. Martin) — authoring skills applied as you write, `reviewing-apex` for existing code.

| Skill | Source | Use when |
|---|---|---|
| `clean-apex-functions` | Ch 3 — Functions | Writing or refactoring methods, handlers, service logic |
| `clean-apex-error-handling` | Ch 7 — Error Handling | Callouts, DML services, exceptions, null handling |
| `reviewing-apex` | Ch 17 + 6, 9, 10 | Reviewing or critiquing existing Apex code |


## Contributing

Each skill lives in `skills/<name>/SKILL.md`.
