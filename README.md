# pi-branch-reviewer

A [pi package](https://github.com/mariozechner/pi-coding-agent) that performs smart code reviews on git branches by classifying changed files and dispatching specialized language/framework reviewers as parallel subagents.

## How It Works

1. **Detects the target branch** — automatically finds `dev`, `develop`, `main`, or `master` (or use a custom target)
2. **Gathers the diff** — collects commit log, file stats, and full diff against the target branch
3. **Classifies files** — maps changed files to the appropriate specialized reviewer(s)
4. **Dispatches reviewers in parallel** — each reviewer gets only its relevant files and diff
5. **Aggregates results** — compiles a structured report with critical issues, warnings, suggestions, and a final recommendation

## Reviewers

| Reviewer | Scope |
|----------|-------|
| **JS/TS** | Modern ES syntax, TypeScript strictness, async/await, null safety, import organization |
| **React** | Hooks correctness, re-render optimization, component patterns, accessibility |
| **PHP** | PHP 8+ features, type safety, PSR-12, error handling, dependency injection |
| **Propel** | Propel ORM patterns, schema.xml, query efficiency, N+1 problems, migration safety |
| **SQL** | Query performance, injection risks, index usage, migration safety, schema design |

## Install

```bash
pi install git:github.com/SunflowerFuchs/pi-subagent-review
```

Or for the current session only:

```bash
pi -e git:github.com/SunflowerFuchs/pi-subagent-review
```

## Usage

Ask pi to review your branch:

```
Review my current branch
Review my branch against main
```

The `branch-review` skill will activate automatically based on the description match.

## What's Included

- **Skill**: `branch-review` — orchestrates the multi-reviewer workflow
- **Agents**: 5 specialized reviewers (`reviewer-js-ts`, `reviewer-react`, `reviewer-php`, `reviewer-propel`, `reviewer-sql`)
