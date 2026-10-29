---
name: branch-review
description: Smart branch review that classifies changed files and dispatches specialized language/framework reviewers as parallel subagents. Use when you need to review changes on the current branch or another specified branch against a target branch.
---

# Smart Branch Review

## Phase 1: Determine Source and Target Branches

Parse the user's request to identify the source and target branches. Typical patterns:
- "review against dev" → source: current branch, target: dev
- "review refactor/LP-1234 against dev" → source: refactor/LP-1234, target: dev
- "review refactor/LP-1234" → source: refactor/LP-1234, target: auto-detect
- No arguments → source: current branch, target: auto-detect

### Resolve Branch References

For both source and target branches, prefer remote references over local ones. Given a branch name (e.g. `dev`), resolve it using this logic:

1. Run `git branch -a --list '<name>' 'remotes/<name>' '*/<name>'` to find matching local and remote branches.
2. If a remote reference exists (e.g. `origin/dev`), use it. Prefer `origin/<name>` over other remotes.
3. If only a local branch exists, use the local name as-is.
4. If the branch name already contains a remote prefix (e.g. `origin/dev`), use it directly.

### Auto-Detect Target Branch

If no target branch was specified by the user, auto-detect it. Run `git branch -a --list 'dev' 'develop' 'main' 'master' '*/*dev' '*/*develop' '*/*main' '*/*master'` to list candidate branches, then pick the first match from this priority list (applying the remote-preference rule above):

1. `origin/dev`
2. `dev`
3. `origin/develop`
4. `develop`
5. `origin/main`
6. `main`
7. `origin/master`
8. `master`

If none of these branches exist, inform the user and stop.

### Source Branch

If no source branch was specified by the user, use the current branch: `SOURCE_BRANCH` = result of `git branch --show-current`. Apply the same remote-preference resolution to the source branch as well.

Use three-dot diff ranges to show only changes on the source branch since it diverged from the target (not changes the target received from other branches). If the source branch is the current branch, the diff range is `<TARGET_BRANCH>...HEAD`. If the source branch is a different branch, the diff range is `<TARGET_BRANCH>...<SOURCE_BRANCH>`.

## Phase 2: Gather Change Overview

Using the `TARGET_BRANCH` and `SOURCE_BRANCH` determined in Phase 1, determine the diff range. **Always use three dots (`...`)** to diff from the merge base, so only the source branch's own changes are included:
- If source is the current branch: `DIFF_RANGE = <TARGET_BRANCH>...HEAD`
- If source is a different branch: `DIFF_RANGE = <TARGET_BRANCH>...<SOURCE_BRANCH>`

Run these git commands:

- `git branch --show-current` — to get the current branch name
- `git log <DIFF_RANGE> --oneline` — to list commits since divergence
- `git diff <DIFF_RANGE> --stat` — for a file change summary
- `git diff <DIFF_RANGE> --name-only` — for the list of changed files

## Phase 3: Classify Changed Files

Analyze the list of changed files and classify them into reviewer buckets. A file can belong to multiple buckets.

**PHP Reviewer** — files matching `*.php`

**Propel Reviewer** — PHP files that are Propel-related. These include:
- Files matching `*Peer.php`, `*Query.php`, `*TableMap.php`
- Files in directories named `map/` or `om/`
- `schema.xml` files
- PHP files containing Propel references (check with grep for `Propel`, `BaseObject`, `BasePeer`, `Criteria`)

**Blade Reviewer** — files matching `*.blade.php` (Blade templates in any `views/`, `resources/views/`, `emails/`, or component directory)

**SCSS / Theming Reviewer** — files matching `*.scss`, `*.css`, and theme-contract markdown (any `*.md` under or near a `themes/` directory, e.g. `THEME*.md`/`COLORS*.md`/tokens specs). A single SCSS file may touch both component defaults and theme overrides — assign the whole file to this reviewer.

**API Contract Reviewer** — files that define or expose the HTTP contract. These include:
- OpenAPI specs: `api.json`, `openapi.json`, `openapi.yaml`, or any `*api*.json` / `*-openapi.*` file
- PHP files in any `Http/Resources/` directory (JSON Resource classes)
- Route files: `routes/api*.php` and any `routes/*.php` explicitly tied to JSON API endpoints
- Generated client type files / directories that mirror the spec (e.g. `types/api/*`, `src/types/api/*` or similar codegen output)
These may overlap with the PHP/JS-TS reviewers; a file can belong to multiple buckets.

**JS/TS Reviewer** — files matching `*.js`, `*.jsx`, `*.ts`, `*.tsx`

**React Reviewer** — JS/TS files that import from `react`, `react-dom`, or contain JSX/TSX component patterns. Check file contents with grep for `from 'react'`, `from "react"`, `React.Component`, `useState`, `useEffect`.

**SQL Reviewer** — files matching `*.sql`, or migration files (files in directories named `migrations/`, `migrate/`, or containing `migration` in the name)

**Rust Reviewer** — files matching `*.rs` (including `build.rs` and `main.rs`/`lib.rs`), and `Cargo.toml` (manifest changes: dependencies, feature flags, edition/MSRV). Exclude `Cargo.lock` — it is generated output.

**General Reviewer (fallback)** — any changed file not claimed by one of the buckets above. This includes sources in languages without a dedicated reviewer, docs, config files, CI pipelines, Dockerfiles, and scripts. Files claimed by a specialized bucket are NOT duplicated into this bucket. Assign files here only after evaluating all specialized buckets; this bucket must be non-empty whenever any changed file matched nothing else.

## Phase 4: Dispatch Reviewer Subagents

For each applicable reviewer bucket, dispatch a subagent **in parallel** using the `subagent` tool. All applicable reviewers MUST be dispatched in a single call with the `tasks` array parameter.

**IMPORTANT:** The main agent MUST NOT read or include the actual diff content. Only pass the target branch reference and file list. The subagent will fetch the diff itself. This keeps the main agent's context clean and avoids wasting tokens on diff content it doesn't need.

For each reviewer subagent, construct a task prompt like this:

```
You are a specialized code reviewer. Review the following changes.

Target branch: <TARGET_BRANCH>
Source branch: <SOURCE_BRANCH> (or HEAD if current branch)
Changed files assigned to you: <FILE_LIST> (space-separated file paths)
Your reviewer tag: <REVIEWER_TAG> (short lowercase id, e.g. php, js-ts, scss)

Your first action MUST be to run: git diff <DIFF_RANGE> -- <FILE_LIST>
Also run: git log <DIFF_RANGE> --oneline -- <FILE_LIST>
Use the output of both commands to perform your review.

Follow the review guidelines from your agent definition. Return your findings in this exact format — one tagged line per finding, ordered critical → warning → suggestion:

[<REVIEWER_TAG>][critical] <finding: what is wrong, where (exact file:line), why it matters, how to fix it>
[<REVIEWER_TAG>][warning] <finding>
[<REVIEWER_TAG>][suggestion] <finding>
[<REVIEWER_TAG>][critical] none
[<REVIEWER_TAG>][warning] none
[<REVIEWER_TAG>][suggestion] none
notes: <1-2 sentences: what you verified and your overall assessment>

Rules:
- One line per finding; for each severity with no findings, emit a single "none" line (no explanation)
- Reference exact file names and line numbers from the diff
- Put verification rationale (what you checked and found clean) in the notes line, never in findings
- No markdown headers, bold, or nested lists — plain tagged lines only
```

Map the reviewer buckets to these subagent agent names:

| Bucket | Agent Name | Tag |
|--------|------------|-----|
| PHP files | `reviewer-php` | `php` |
| Propel files | `reviewer-propel` | `propel` |
| Blade templates | `reviewer-blade` | `blade` |
| SCSS / theming files | `reviewer-scss-theming` | `scss` |
| API contract files | `reviewer-api-contract` | `api` |
| JS/TS files | `reviewer-js-ts` | `js-ts` |
| React files | `reviewer-react` | `react` |
| SQL files | `reviewer-sql` | `sql` |
| Rust files | `reviewer-rust` | `rust` |
| Leftover/unclassified files | `reviewer-general` | `general` |

Use the parallel dispatch pattern:

```
subagent({
  tasks: [
    { agent: "reviewer-php", task: "<constructed prompt>" },
    { agent: "reviewer-react", task: "<constructed prompt>" }
  ]
})
```

## Phase 5: Aggregate Results

Once all subagent reviewers have returned their results, merge them into a single severity-first report. The merge is mechanical — do not reword, re-rank, or drop findings:

- Collect every `[tag][severity]` line from all reviewers and drop their `none` lines
- Group by severity: all critical findings first, then warnings, then suggestions
- Within each severity group, keep reviewer dispatch order
- Each reviewer's `notes:` line becomes one bullet under Reviewer notes
- If a reviewer returned nonconforming output (free-form prose instead of tagged lines), map its points into tagged findings at the fitting severity and put the remainder in its Reviewer notes bullet

Format every finding as `<n>. **[<tag>]** <finding text>` — the tag stays bold at the start of the entry, numbered continuously within each severity group.

Compile the final review in this format:

---

# Branch Review: `<SOURCE_BRANCH>` → `<TARGET_BRANCH>`

## Overview
- **Source**: `<SOURCE_BRANCH>` (or current branch if HEAD)
- **Target**: `<TARGET_BRANCH>`
- **Commits**: <number of commits>
- **Files changed**: <number of files>

## Critical

<numbered list of all critical findings across all reviewers, or "None across all reviewers.">

## Warnings

<numbered list of all warnings across all reviewers, or "None across all reviewers.">

## Suggestions

<numbered list of all suggestions across all reviewers, or "None across all reviewers.">

## Reviewer notes

- **<tag>** — <the reviewer's notes line> (one bullet per reviewer, in dispatch order)

## Summary

- **Top concerns**: <list the most critical findings across all reviewers>
- **Overall assessment**: <brief overall assessment>
- **Recommendation**: <one of: "Approve", "Approve with minor changes", "Request changes">

Base recommendation on:
- Any critical issue → "Request changes"
- Warnings but no critical → "Approve with minor changes"
- Only suggestions or clean → "Approve"

---
