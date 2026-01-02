# UKV Code Review Slash Command

You are the orchestrator for a thorough, context-efficient code review. This
command uses a hybrid approach with categorized sub-agents and file-based
intermediate storage to stay within context headroom.

## Parameters

Parse the command arguments:

- Format: `/ukv-review [PR_ID_or_URL]`
- **PR_ID_or_URL** (optional): GitHub PR ID or full URL
  - If provided: Review the specified PR
  - If omitted: Review local commits not yet pushed to remote

**Examples:**

- `/ukv-review` → Review local unpushed commits
- `/ukv-review 123` → Review PR #123
- `/ukv-review https://github.com/clamshell-ai/humor/pull/123` → Review PR #123

## Architecture Overview

This review spawns **4 parallel category sub-agents** instead of 11 individual
agents. Each category agent:

1. Analyzes multiple related dimensions together (enables cross-referencing)
2. Writes findings to a temp file in `ai/docs/temp-reviews/`
3. Returns completion status

After all agents complete, you read each temp file with context compaction
between reads, then synthesize the final review.

```
┌─────────────────────────────────────────────────────────────────┐
│                     UKV-REVIEW ORCHESTRATOR                     │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Code Quality   │ │  Architecture   │ │    Testing &    │
│    & Style      │ │    & Design     │ │   Reliability   │
│   (4 dims)      │ │   (3 dims)      │ │   (2 dims)      │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ code-quality-   │ │ architecture-   │ │ testing-        │
│ {PR_ID}.md      │ │ {PR_ID}.md      │ │ reliability-    │
│                 │ │                 │ │ {PR_ID}.md      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Ops & Perf      │
                    │ (2 dims)        │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ ops-performance-│
                    │ {PR_ID}.md      │
                    └─────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              SYNTHESIZE → ai/docs/reviews/review-{PR_ID}.md     │
└─────────────────────────────────────────────────────────────────┘
```

## Instructions

### Step 1: Setup and Scope Determination

1. **Determine PR context:**
   - If PR ID/URL provided: Extract PR number, fetch PR details with
     `gh pr view`
   - If no PR: Get local unpushed commits with
     `git log origin/$(git branch --show-current)..HEAD`

2. **Identify the changeset:**
   - For PR: `gh pr diff {PR_ID}`
   - For local: `git diff origin/$(git branch --show-current)...HEAD`

3. **Create output directories if needed:**

   ```bash
   mkdir -p ai/docs/temp-reviews ai/docs/reviews
   ```

4. **Determine PR_ID for filenames:**
   - If PR: Use PR number (e.g., `123`)
   - If local: Use branch name sanitized (e.g., `feature-foo`)

5. **Show scope to user:**
   - List changed files
   - Identify affected components (janomus, hermelos, pundora, web, functions,
     k8s)
   - Confirm proceeding with review

### Step 2: Spawn Category Sub-Agents (Parallel)

Spawn **all 4 category agents in parallel** using the Task tool with
`subagent_type: general-purpose`. Each agent must:

- Read all relevant CLAUDE.md files for affected components
- Analyze the full changeset for its assigned dimensions
- Cross-reference findings within its category
- Write findings to its designated temp file
- Group issues by severity: CRITICAL > HIGH > MEDIUM > LOW

---

#### Category 1: Code Quality & Style

**Temp File:** `ai/docs/temp-reviews/code-quality-{PR_ID}.md`

**Prompt for sub-agent:**

```
You are reviewing PR/changeset {PR_ID} for CODE QUALITY & STYLE issues.

First, read all relevant CLAUDE.md files for affected components.
Then analyze the changeset for these 4 dimensions:

1. **Inconsistencies & Accuracy**
   - Look at the current code for any inconsistencies
   - Ensure accuracy of implementation vs. intent
   - Check for logical errors or incorrect assumptions

2. **Code Smells & Terseness**
   - Identify code smells and anti-patterns
   - Make implementation more terse where applicable
   - Flag unnecessary complexity or verbosity

3. **Naming Conventions**
   - Verify function names follow project conventions
   - Check test function names are descriptive
   - Ensure variable/constant naming is consistent

4. **Business Logic Verbosity**
   - Verify business logic is not overly verbose
   - Suggest improvements to make code more concise
   - Identify redundant or duplicated logic

Cross-reference findings across dimensions. Group related issues.

Write your findings to: ai/docs/temp-reviews/code-quality-{PR_ID}.md

Format:
## Code Quality & Style Review

### CRITICAL Issues
- [Dimension] Description - file:line

### HIGH Priority Issues
- [Dimension] Description - file:line

### MEDIUM Priority Issues
- [Dimension] Description - file:line

### LOW Priority Issues
- [Dimension] Description - file:line

### Cross-References
- Note where issues in one dimension relate to another

Include specific file paths and line numbers for all findings.
```

---

#### Category 2: Architecture & Design

**Temp File:** `ai/docs/temp-reviews/architecture-{PR_ID}.md`

**Prompt for sub-agent:**

```
You are reviewing PR/changeset {PR_ID} for ARCHITECTURE & DESIGN issues.

First, read all relevant CLAUDE.md files for affected components.
Also read any research/plan documents referenced in the PR or commits.
Then analyze the changeset for these 3 dimensions:

1. **Documentation Currency**
   - Verify all CLAUDE.md files updated to reflect functionality changes
   - Check if new patterns/conventions need documentation
   - Ensure API changes are documented

2. **Guideline Adherence**
   - Check for deviations from CLAUDE.md guidelines
   - Ensure configurable values are NOT hardcoded constants
   - Verify SOLID principles are followed
   - Check dependency injection patterns (no lazy singletons outside main.go)

3. **Hindsight Review**
   - Review the original Linear ticket (if referenced)
   - Compare against research/plan documents
   - If implementing from scratch with benefit of hindsight:
     - Would you recommend a different approach?
     - Are there more pragmatic alternatives?
   - Share alternatives with clear recommendations

Cross-reference findings. Note where design choices impact multiple areas.

Write your findings to: ai/docs/temp-reviews/architecture-{PR_ID}.md

Format:
## Architecture & Design Review

### CRITICAL Issues
- [Dimension] Description - file:line

### HIGH Priority Issues
- [Dimension] Description - file:line

### MEDIUM Priority Issues
- [Dimension] Description - file:line

### LOW Priority Issues
- [Dimension] Description - file:line

### Alternative Approaches (Hindsight)
- [Approach] Description with pros/cons
- Recommendation: ...

### Cross-References
- Note where design issues cascade to other areas

Include specific file paths and line numbers for all findings.
```

---

#### Category 3: Testing & Reliability

**Temp File:** `ai/docs/temp-reviews/testing-reliability-{PR_ID}.md`

**Prompt for sub-agent:**

```
You are reviewing PR/changeset {PR_ID} for TESTING & RELIABILITY issues.

First, read all relevant CLAUDE.md files for affected components.
Read pkg/CLAUDE.md for concurrency patterns (CRITICAL for race conditions).
Then analyze the changeset for these 2 dimensions:

1. **Test Coverage Gaps**
   - Identify gaps in test coverage for changed code
   - Recommend additions using:
     - Unit tests (standard go test / pytest / vitest)
     - Firebase emulator-based tests (Firestore, Pub/Sub)
     - Docker-based per-service integration tests
   - Check if edge cases are covered
   - Verify error paths are tested

2. **Correctness & Security**
   - Check research, plan, AND implementation for:
     - Implementation bugs
     - Race conditions (especially in Go - check mutex usage, channel patterns)
     - Security vulnerabilities:
       - Generic: SQL injection, XSS, command injection, path traversal
       - Domain-specific:
         - Matrix MXID validation (@ prefix, :domain suffix)
         - MULP validation (no @ or :)
         - Firestore query parameterization
         - GCP Secret Manager (never log secrets)
         - Firebase token verification
         - Bridge credentials (encrypted, never logged)
         - Rate limiting for Matrix API
         - Authorization checks

Cross-reference findings. Note where missing tests could catch identified bugs.

Write your findings to: ai/docs/temp-reviews/testing-reliability-{PR_ID}.md

Format:
## Testing & Reliability Review

### CRITICAL Issues
- [Dimension] Description - file:line

### HIGH Priority Issues
- [Dimension] Description - file:line

### MEDIUM Priority Issues
- [Dimension] Description - file:line

### LOW Priority Issues
- [Dimension] Description - file:line

### Recommended Test Additions
- [Type: unit/emulator/integration] Test description - for file:line

### Cross-References
- Note where test gaps relate to identified bugs/vulnerabilities

Include specific file paths and line numbers for all findings.
```

---

#### Category 4: Operations & Performance

**Temp File:** `ai/docs/temp-reviews/ops-performance-{PR_ID}.md`

**Prompt for sub-agent:**

```
You are reviewing PR/changeset {PR_ID} for OPERATIONS & PERFORMANCE issues.

First, read all relevant CLAUDE.md files for affected components.
Then analyze the changeset for these 2 dimensions:

1. **Logging & Metrics**
   - Review logging quality and completeness
   - Emphasize structured logging (key-value pairs, not string interpolation)
   - Ensure relevant fields included (request IDs, user IDs, timestamps)
   - Verify metrics are tracked where useful:
     - Counters for operations
     - Histograms for latencies
     - Gauges for current state
   - Check log levels are appropriate (debug vs info vs error)

2. **Performance Optimizations**
   - Suggest optimizations for:
     - Database queries (N+1, missing indexes, unnecessary reads)
     - Network calls (batching, caching, connection pooling)
     - Marshalling/unmarshalling (avoid repeated parsing)
     - Data structure choices (maps vs slices, appropriate sizing)
     - Algorithm complexity (O(n²) that could be O(n))
     - Memory usage (unnecessary allocations, large copies)
   - Identify hot paths that need optimization
   - Flag premature optimization (keep it simple where perf isn't critical)

Cross-reference findings. Note where logging could help diagnose perf issues.

Write your findings to: ai/docs/temp-reviews/ops-performance-{PR_ID}.md

Format:
## Operations & Performance Review

### CRITICAL Issues
- [Dimension] Description - file:line

### HIGH Priority Issues
- [Dimension] Description - file:line

### MEDIUM Priority Issues
- [Dimension] Description - file:line

### LOW Priority Issues
- [Dimension] Description - file:line

### Logging Recommendations
- [Add/Modify] Description - file:line

### Performance Recommendations
- [Optimization type] Description - file:line - Expected impact

### Cross-References
- Note where logging relates to performance debugging

Include specific file paths and line numbers for all findings.
```

---

### Step 3: Collect and Synthesize Results

After all 4 agents complete:

1. **Read each temp file with context compaction between reads:**
   - Read `ai/docs/temp-reviews/code-quality-{PR_ID}.md`
   - Compact context (focus on actionables)
   - Read `ai/docs/temp-reviews/architecture-{PR_ID}.md`
   - Compact context
   - Read `ai/docs/temp-reviews/testing-reliability-{PR_ID}.md`
   - Compact context
   - Read `ai/docs/temp-reviews/ops-performance-{PR_ID}.md`

2. **Synthesize findings into final review:**
   - Deduplicate issues that appear in multiple categories
   - Cross-reference related findings across categories
   - Prioritize by severity and impact
   - Group by actionability

3. **Write final review to:** `ai/docs/reviews/review-{PR_ID}.md`

### Step 4: Write Final Review

Create `ai/docs/reviews/review-{PR_ID}.md` with this format:

```markdown
# Code Review: {PR_ID}

**Date:** {date}
**Reviewer:** Claude (ukv-review)
**Changeset:** {description}

## Executive Summary

{2-3 sentence summary of overall code quality and key concerns}

- Total Issues: X (Y critical, Z high, W medium, V low)
- Key Concerns: {list top 3}
- Recommendation: {approve / request changes / needs discussion}

## Critical Issues

{Must be fixed before merge}

### Issue 1: {title}

- **Category:** {category}
- **Location:** `file:line`
- **Description:** {what's wrong}
- **Recommendation:** {how to fix}

## High Priority Issues

{Should be fixed before merge}

### Issue 1: {title}

...

## Medium Priority Issues

{Consider fixing, can be follow-up}

...

## Low Priority Issues

{Nice to have, optional}

...

## Alternative Approaches (Hindsight Review)

{If applicable, include hindsight recommendations}

## Recommended Test Additions

{Specific tests to add}

## Cross-Cutting Concerns

{Issues that span multiple categories}

---

_This review was generated by `/ukv-review`. Do not edit this file directly._
```

### Step 5: Cleanup

After final review is written:

1. **Delete temp files:**

   ```bash
   rm -f ai/docs/temp-reviews/code-quality-{PR_ID}.md
   rm -f ai/docs/temp-reviews/architecture-{PR_ID}.md
   rm -f ai/docs/temp-reviews/testing-reliability-{PR_ID}.md
   rm -f ai/docs/temp-reviews/ops-performance-{PR_ID}.md
   ```

2. **Report completion to user:**
   - Show path to final review file
   - Show executive summary
   - Show count of issues by severity

## Important Notes

- **Always use Task tool** to spawn sub-agents (never do review work directly)
- **Do NOT make code changes** or update the PR - this is review only
- **Include file:line references** for all findings
- **Focus on actionables** - avoid noise and nitpicks
- **Cross-reference** findings across categories for comprehensive analysis
- **Context compaction** is critical when reading temp files - focus on
  actionables
- **Temp files** are intermediate artifacts - always clean up after synthesis
- **Final review** goes in `ai/docs/reviews/` for permanent record
