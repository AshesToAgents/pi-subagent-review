---
name: branch-review
description: Smart branch review that classifies changed files and dispatches specialized language/framework reviewers as parallel subagents. Use when you need to review changes on the current branch against a target branch.
---

# Smart Branch Review

## Phase 1: Determine Target Branch

If the user provided a target branch argument, use that as `TARGET_BRANCH`.

Otherwise, auto-detect the target branch. Run `git branch -a --list 'dev' 'develop' 'main' 'master' '*/*dev' '*/*develop' '*/*main' '*/*master'` to only list candidate branches, then pick the first match from this priority list:

1. `origin/dev`
2. `dev`
3. `origin/develop`
4. `develop`
5. `origin/main`
6. `main`
7. `origin/master`
8. `master`

If none of these branches exist, inform the user and stop.

## Phase 2: Gather Change Overview

Using the `TARGET_BRANCH` determined in Phase 1, run these git commands:

- `git branch --show-current` — to get the current branch name
- `git log <TARGET_BRANCH>..HEAD --oneline` — to list commits since divergence
- `git diff <TARGET_BRANCH>..HEAD --stat` — for a file change summary
- `git diff <TARGET_BRANCH>..HEAD --name-only` — for the list of changed files

## Phase 3: Classify Changed Files

Analyze the list of changed files and classify them into reviewer buckets. A file can belong to multiple buckets.

**PHP Reviewer** — files matching `*.php`

**Propel Reviewer** — PHP files that are Propel-related. These include:
- Files matching `*Peer.php`, `*Query.php`, `*TableMap.php`
- Files in directories named `map/` or `om/`
- `schema.xml` files
- PHP files containing Propel references (check with grep for `Propel`, `BaseObject`, `BasePeer`, `Criteria`)

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
Changed files assigned to you: <FILE_LIST> (space-separated file paths)

Your first action MUST be to run: git diff <TARGET_BRANCH>..HEAD -- <FILE_LIST>
Also run: git log <TARGET_BRANCH>..HEAD --oneline -- <FILE_LIST>
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
|--------|-----------|
| PHP files | `reviewer-php` |
| Propel files | `reviewer-propel` |
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

# Branch Review: `<current_branch>` → `<TARGET_BRANCH>`

## Overview
- **Branch**: `<current_branch>`
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
