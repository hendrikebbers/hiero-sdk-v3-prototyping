---
name: spec-flow
license: Apache-2.0
metadata:
  source: https://github.com/open-elements/claude-base
  author: Open Elements
description: Implement a completed spec end-to-end using a proper GitHub flow. Creates a GitHub issue (if missing), checks out a feature branch, runs spec-implement, iteratively reviews with spec-review and quality-review until clean, and opens a Pull Request. Use this skill when the user has a finished spec (design.md + behaviors.md) and wants to turn it into a mergeable PR with full CI-friendly branching — or when the user says "implement this spec", "create a PR for this spec", or "run the spec flow".
---

# Spec Flow — From Spec to Pull Request

Take a completed specification and turn it into a reviewed, ready-to-merge Pull Request using GitHub flow. This skill orchestrates the full lifecycle: issue tracking, branching, implementation, iterative review, and PR creation.

Before starting, read `../../conventions/spec-driven-development.md` for the spec folder structure and conventions.

## Prerequisites

- A completed spec with `design.md` and `behaviors.md` in the `docs/specs/` directory
- The `gh` CLI must be available and authenticated
- The working tree must be clean (no uncommitted changes on the current branch)

## Instructions

### 1. Identify the spec

Ask the user which spec to implement, or detect it from context. If no spec is specified, check `docs/specs/INDEX.md` for specs with status `open` — these are candidates.

Read both `design.md` and `behaviors.md` from the spec folder to confirm it is complete enough to implement. If either file is missing or clearly incomplete, tell the user and suggest running `/spec-create` first.

### 2. Ensure a GitHub issue exists

Check the spec's entry in `docs/specs/INDEX.md` for a GitHub issue reference.

**If an issue is linked** (e.g., `#42`), verify it exists using `gh issue view <number>`. If it does, continue.

**If no issue exists** (`—` in the index), create one:

1. Derive the issue title from the spec name
2. Write the issue body with:
   - A summary from `design.md`
   - A link to the spec folder
   - Key behavioral scenarios from `behaviors.md` as acceptance criteria
3. Create the issue: `gh issue create --title "..." --body "..."`
4. Update `docs/specs/INDEX.md` to record the new issue number

### 3. Create a feature branch

Determine an appropriate branch name from the spec: `feat/<spec-folder-name>` (e.g., `feat/001-user-auth-flow`). For bug fix specs, use `fix/` instead of `feat/`.

```bash
git checkout -b <branch-name>
```

If the branch already exists (e.g., from a previous attempt), ask the user whether to reuse it or create a fresh one.

### 4. Implement the spec

Update the spec's status to `in progress` in `docs/specs/INDEX.md` and commit this change.

Invoke `/spec-implement` to generate the implementation plan and execute it. This means:

1. Invoke the `spec-implement` skill
2. Follow its instructions to create `steps.md` and work through the implementation steps
3. Use automated execution mode — work through steps sequentially, checking off items as they are completed
4. After each step, verify the project builds and tests pass before moving on

If a build or test failure occurs, diagnose and fix it before proceeding to the next step.

### 5. Review and iterate

After implementation is complete, run two review passes. Repeat this cycle until both reviews are clean.

#### 5a. Spec review

Invoke the `spec-review` skill and follow its instructions to review the implementation against the spec. This checks:

- Every element in `design.md` is implemented
- Every scenario in `behaviors.md` has a corresponding test
- No gaps between spec and code

#### 5b. Quality review

Invoke the `quality-review` skill and follow its instructions to review the code for quality. This checks:

- Code quality (DRY, KISS, naming, dead code)
- Security (no hardcoded secrets, input validation, parameterized queries)
- Test quality (coverage, determinism, edge cases)
- Convention compliance (language-specific rules, architecture patterns)

#### 5c. Fix issues

If either review surfaces **Critical** or **Improvement** findings:

1. Summarize the findings to the user
2. Fix the identified issues
3. Run the build and tests to confirm the fixes are correct
4. Re-run both reviews (5a and 5b)

Repeat until:
- The spec review shows full coverage (all design elements covered, all scenarios tested)
- The quality review shows no Critical or Improvement findings

**Suggestions** from the quality review are optional — present them to the user but do not block on them.

After each fix cycle, commit the changes with a descriptive message referencing the spec.

### 6. Final commit and push

Ensure all changes are committed. Update `docs/specs/INDEX.md` to set the spec status to `done` and commit this separately.

Push the feature branch to the remote:

```bash
git push -u origin <branch-name>
```

### 7. Create the Pull Request

Create a PR using the `gh` CLI. The PR should reference the GitHub issue so it closes automatically on merge.

```bash
gh pr create --title "<short title>" --body "$(cat <<'EOF'
## Summary

<2-3 sentence summary from design.md>

## Spec

- Spec folder: `docs/specs/<spec-folder>/`
- Design: `docs/specs/<spec-folder>/design.md`
- Behaviors: `docs/specs/<spec-folder>/behaviors.md`

## Changes

<bulleted list of key changes>

## Test coverage

- <count> behavioral scenarios covered
- All spec-review checks pass
- All quality-review checks pass

Closes #<issue-number>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### 8. Verify CI checks

After the PR is created, check whether the repository has GitHub Actions workflows. If it does, wait for the CI checks to complete:

1. Poll the PR check status using `gh pr checks <pr-number> --watch` (times out after 5 minutes)
2. If all checks pass, continue to the summary
3. If checks fail within 5 minutes:
   - Read the failing workflow logs: `gh run view <run-id> --log-failed`
   - Diagnose the failure — common causes include test failures, lint errors, build issues, or environment mismatches between local and CI
   - Fix the issue, commit, and push to the branch
   - Wait for the new CI run (again up to 5 minutes)
   - If checks still fail after one fix attempt, report the failure details to the user and ask how to proceed rather than looping indefinitely
4. If no checks appear within 2 minutes, the repository likely has no CI configured — note this in the summary and move on

### 9. Summary

Present the final result to the user:

- Link to the Pull Request
- Link to the GitHub issue
- Number of commits on the branch
- Number of behavioral scenarios covered
- Any deferred suggestions from the quality review
- Remind the user to request a human code review before merging

## Behavioral Rules

- **Clean working tree required** — Do not start if there are uncommitted changes. Ask the user to commit or stash first.
- **One spec, one branch, one PR** — Each spec gets its own feature branch and PR. Do not combine multiple specs.
- **Iterative reviews are non-negotiable** — Do not create the PR until both spec-review and quality-review are satisfied. The goal is a PR that is ready for human review, not one that needs further automated fixes.
- **User stays informed** — Report progress at key milestones (branch created, implementation done, review results, PR created). Do not go silent during long implementation phases.
- **Respect existing work** — If the feature branch already has commits, ask before resetting or force-pushing.
- **Commits are granular** — Create meaningful commits during implementation, not one giant commit at the end. Each implementation step or fix cycle should be its own commit.
