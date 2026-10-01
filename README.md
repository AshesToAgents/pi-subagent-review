# pi-subagent-review

A [pi](https://github.com/earendil-works/pi) package that performs smart code reviews by classifying changed files and dispatching specialized reviewers as parallel subagents.

## Install

```bash
# Global (user-level)
pi install ssh://git@github.com/AshesToAgents/pi-subagent-review.git

# Project-level (shared with team via .pi/settings.json)
pi install -l ssh://git@github.com/AshesToAgents/pi-subagent-review.git

# Try without installing
pi -e ssh://git@github.com/AshesToAgents/pi-subagent-review.git
```

## What's Included

| Type | Name | Description |
|------|------|-------------|
| Skill | `branch-review` | Orchestrates the multi-reviewer workflow |
| Agent | `reviewer-js-ts` | JS/TS: modern syntax, TypeScript strictness, async/await, null safety, imports |
| Agent | `reviewer-react` | React: hooks correctness, re-render optimization, component patterns, accessibility |
| Agent | `reviewer-php` | PHP: PHP 8+ features, type safety, PSR-12, error handling, dependency injection |
| Agent | `reviewer-propel` | Propel ORM: schema.xml, query efficiency, N+1 problems, migration safety |
| Agent | `reviewer-sql` | SQL: query performance, injection risks, index usage, migration safety, schema design |
| Agent | `reviewer-blade` | Blade: output escaping/XSS, directive correctness, render-mode integrity, stale relation calls |
| Agent | `reviewer-scss-theming` | SCSS/theming: custom-property role tokens, var(--token, fallback) contracts, theme-vs-component specificity |
| Agent | `reviewer-api-contract` | API contract: route→controller→Resource→OpenAPI→generated-TS symmetry, typed Resource shapes, drift |
| Agent | `reviewer-rust` | Rust: memory safety, panic avoidance, async correctness, unsafe code scrutiny, idiomatic patterns, Cargo.toml deps |

## Usage

Ask pi to review your branch:

```
Review my current branch
Review my branch against main
```

The `branch-review` skill activates automatically based on the description match.

## How It Works

1. **Detects the target branch** — automatically finds `dev`, `develop`, `main`, or `master` (or use a custom target)
2. **Gathers the diff** — collects commit log, file stats, and full diff against the target branch
3. **Classifies files** — maps changed files to the appropriate specialized reviewer(s)
4. **Dispatches reviewers in parallel** — each reviewer gets only its relevant files and diff
5. **Aggregates results** — compiles a structured report with critical issues, warnings, suggestions, and a final recommendation

## Development

```bash
npm install
npm run typecheck
npm test
```

## License

MIT
