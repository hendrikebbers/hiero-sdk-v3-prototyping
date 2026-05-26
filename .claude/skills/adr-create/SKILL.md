---
name: adr-create
license: Apache-2.0
metadata:
  source: https://github.com/open-elements/claude-base
  author: Open Elements
description: Create an Architecture Decision Record (ADR) that documents one architecturally significant decision with full justification. Mandatorily stress-tests the decision with /grill-me first, then records context, the chosen option, rejected alternatives, and consequences — where every factual claim must be backed by a verifiable source. Use this skill when the user wants to record an architectural decision, document a technology or library choice, justify rejecting a dependency, capture a design trade-off, or write/supersede an ADR. Use this skill when the user does a clear architectural decision - in that case suggest the creation of a new ADR.  
argument-hint: [the architectural decision, technology choice, or trade-off to record]
---

# Create Architecture Decision Record (ADR)

Record **one** architecturally significant decision as an ADR following Michael Nygard's template. An ADR is the durable answer to "why is it built this way?" — it must therefore be honest, sourced, and complete enough to survive being read by someone who wasn't in the room.

This skill is built around two non-negotiable rules: the decision is **grilled before it is written**, and **every factual claim carries a source**. A decision recorded without evidence is just an opinion with a date on it.

## Evidence Rules (Never Break These)

1. **No unsourced facts.** Every assertion in *Context* and *Consequences* — versions, dependency counts, performance numbers, project-maturity signals, capability claims — must trace to a verifiable source. State the source, not just the fact.
2. **Verify, do not recall.** Do not state a dependency version, transitive tree, API capability, or benchmark from memory. Read the actual build file, fetch the official docs, query the registry, run the measurement.
3. **An estimate is not a fact.** If a number cannot be measured or cited, label it explicitly as an estimate or assumption — never present it as established.
4. **Alternatives must be real.** A decision with no rejected alternative is not a decision. Name what was considered and why it lost, with the same sourcing standard.
5. **Be honest about the downside.** An ADR with only positive consequences is incomplete. Every decision has costs — record them.
6. **One decision per ADR.** If the input bundles several decisions, split them into separate ADRs.

## Instructions

### 1. Gather the decision

Accept as a starting point: a free-text description, a GitHub issue, a `design.md` from `/spec-create`, or the output of a prior `/grill-me` session.

Clarify with the user **what single decision** is being recorded. An ADR is warranted when the decision is *architecturally significant* — it shapes structure, introduces or rejects a dependency, defines an interface, or is costly to reverse. If the decision is trivial or easily reversible, say so and ask whether an ADR is really needed.

State the decision back to the user in one sentence to confirm scope. If multiple decisions are entangled, propose splitting them into separate ADRs before continuing.

### 2. Grill the decision (mandatory)

Invoke `/grill-me` with the decision as input. This step is **not optional** — do not write the ADR until the grill has run.

The grill operates in the **Implementation** domain. Make sure it drives out, at minimum:
- The real **problem and forces** — what pressure makes this decision necessary now?
- The **alternatives** considered and the concrete reason each was rejected.
- The **negative consequences** and ownership costs, not just the benefits.
- **Reversibility** — how hard is it to undo this later, and what's the exit signal?
- **Who and what is affected** — blast radius across the codebase and team.

When the grill concludes, carry its resolved decisions, surfaced assumptions, and open tensions forward into the ADR. Anything the grill flagged as an assumption must be sourced in the next step or recorded explicitly as an assumption.

### 3. Build the evidence base

Before writing a single line of the ADR, gather and **verify** the sources for every claim the decision rests on. Maintain a working list mapping each claim → its source. Match the source type to the claim:

| Claim type | How to source it |
|------------|------------------|
| **Codebase facts** (patterns, existing wrappers, current usage) | Read the actual files. Cite `path:line`. |
| **Dependency / version facts** (versions, transitive tree, conflicts) | Read the build file (`pom.xml`, `build.gradle`, `package.json`, lockfile). Verify versions against the registry — e.g. the `maven-central` MCP tools for Java artifacts. Never quote a version from memory. |
| **Library / framework capabilities** | Fetch the **official** documentation with `WebFetch` and cite the URL. |
| **Third-party project health** (maturity, release cadence, open issues, stars) | Fetch the actual repository or registry page. Cite it **with the access date** — these numbers age. |
| **Performance / benchmark numbers** | Use a measurement you ran or a citable published source. If neither exists, label it an estimate (Evidence Rule 3). |
| **Standards / regulations** (e.g. GDPR, licensing) | Cite the specific clause, license identifier, or official text. |

Any claim that survives this step without a source must be **dropped or demoted to an explicit assumption**. Present the evidence base to the user briefly so gaps are visible before writing.

### 4. Locate the ADR directory and assign a number

Find the project's ADR directory — conventionally `docs/adr/`. If none exists, propose creating `docs/adr/` together with a `README.md` index (template below) and confirm with the user.

Read the existing ADRs to find the highest sequential number. The new ADR's number is that value `+ 1`, **zero-padded to four digits**. The filename is `NNNN-kebab-case-title.md` (e.g. `0002-adopt-restclient-wrapper.md`) — one decision per file.

### 5. Write the ADR

Create `docs/adr/NNNN-kebab-case-title.md` using this template. Keep it short — if it runs past roughly two screens, link out instead of embedding.

```markdown
# ADR-NNNN: <decision title>

**Status:** Proposed
**Date:** YYYY-MM-DD

## Context

<The problem and the forces at play. State the constraints, the current
situation, and what's pushing the decision. Every factual claim here is
sourced — inline where natural, and listed in References below.>

## Decision

<What is chosen, stated plainly and actively ("We will…" / "The team
rejects…"). State explicitly what is NOT chosen and why the leading
alternatives were rejected.>

## Consequences

### Positive
- <Benefit — sourced or self-evidently following from the decision>

### Negative
- <Cost, risk, or ownership burden the team now accepts>

### Follow-ups
- <Concrete actions, and the condition under which this decision
  should be revisited>

---

**References:** <every source backing the claims above — file paths with
lines, doc URLs, registry pages with access dates, build coordinates,
license identifiers>
```

Rules for the content:
- **Status** starts as `Proposed` unless the user states the decision is already made and agreed, in which case use `Accepted`. Allowed values: `Proposed`, `Accepted`, `Deprecated`, `Superseded by ADR-NNNN`.
- **Date** is today's date in `YYYY-MM-DD`.
- Use `###` sub-headings inside *Context* to group distinct forces (e.g. "Existing stack constraints", "Dependencies introduced", "Quality assessment") when the context is non-trivial.
- The **References** section is mandatory and must let a reader independently verify each claim. A claim in the body with no corresponding source is a defect — fix it before finishing.

### 6. Handle superseding (if applicable)

If this decision replaces an earlier one, **do not edit the old ADR's body** — ADRs are an immutable log. Instead:
1. Change the old ADR's `Status` to `Superseded by ADR-NNNN`.
2. Reference the old ADR from the new one's *Context*, summarizing what changed and why.

### 7. Update the index

Add the new ADR to `docs/adr/README.md`. If the README does not exist, create it:

```markdown
# Architecture Decision Records

This directory records architecturally significant decisions using
[Michael Nygard's template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

- Files are named `NNNN-kebab-case-title.md` with zero-padded sequential numbers.
- One decision per file. ADRs are immutable: never edit a past record —
  supersede it with a new ADR instead.
- Keep ADRs short. If you need more than two screens, link out.

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-NNNN](NNNN-kebab-case-title.md) | <title> | Proposed | YYYY-MM-DD |
```

When superseding, also update the old ADR's row status to `Superseded by ADR-NNNN`.

### 8. Summary

After writing the files, provide:
- A link to the created ADR and its current status.
- The list of sources cited, and any claims that remain explicit assumptions (Evidence Rule 3) so the user can close them.
- The next step: circulate the ADR for review and, once agreed, flip the status from `Proposed` to `Accepted`.