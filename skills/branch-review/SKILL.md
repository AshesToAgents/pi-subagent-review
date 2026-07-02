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

If no files match any reviewer bucket, inform the user that no specialized reviewers are applicable and provide a brief general summary of the changes instead.

## Phase 4: Dispatch Reviewer Subagents

For each applicable reviewer bucket, dispatch a subagent **in parallel** using the `subagent` tool. All applicable reviewers MUST be dispatched in a single call with the `tasks` array parameter.

**IMPORTANT:** The main agent MUST NOT read or include the actual diff content. Only pass the target branch reference and file list. The subagent will fetch the diff itself. This keeps the main agent's context clean and avoids wasting tokens on diff content it doesn't need.

For each reviewer subagent, construct a task prompt like this:

```
You are a specialized code reviewer. Review the following changes.

Target branch: <TARGET_BRANCH>
Source branch: <SOURCE_BRANCH> (or HEAD if current branch)
Changed files assigned to you: <FILE_LIST> (space-separated file paths)

Your first action MUST be to run: git diff <DIFF_RANGE> -- <FILE_LIST>
Also run: git log <DIFF_RANGE> --oneline -- <FILE_LIST>
Use the output of both commands to perform your review.

Follow the review guidelines from your agent definition. Return your findings in this exact format:

## <Reviewer Name> Review

### Critical Issues
<numbered list, or "None found.">

### Warnings
<numbered list, or "None found.">

### Suggestions
<numbered list, or "None found.">

### Summary
<1-2 sentence overall assessment from this reviewer's perspective>
```

Map the reviewer buckets to these subagent agent names:

| Bucket | Agent Name |
|--------|------------|
| PHP files | `reviewer-php` |
| Propel files | `reviewer-propel` |
| Blade templates | `reviewer-blade` |
| SCSS / theming files | `reviewer-scss-theming` |
| API contract files | `reviewer-api-contract` |
| JS/TS files | `reviewer-js-ts` |
| React files | `reviewer-react` |
| SQL files | `reviewer-sql` |

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

Once all subagent reviewers have returned their results, compile the final review in this format:

---

# Branch Review: `<SOURCE_BRANCH>` → `<TARGET_BRANCH>`

## Overview
- **Source**: `<SOURCE_BRANCH>` (or current branch if HEAD)
- **Target**: `<TARGET_BRANCH>`
- **Commits**: <number of commits>
- **Files changed**: <number of files>

## Reviewer Reports

<Insert each reviewer's section here, in order: Critical issues first, then Warnings, then Suggestions>

## Summary

- **Top concerns**: <list the most critical findings across all reviewers>
- **Overall assessment**: <brief overall assessment>
- **Recommendation**: <one of: "Approve", "Approve with minor changes", "Request changes">

Base recommendation on:
- Any critical issue → "Request changes"
- Warnings but no critical → "Approve with minor changes"
- Only suggestions or clean → "Approve"

---
