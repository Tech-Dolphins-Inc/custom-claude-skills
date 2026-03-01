# PR Iterate

Automated PR iteration loop that runs local checks, fixes issues, pushes,
monitors CI, fetches Claude bot review feedback, and applies fixes — repeating
until the PR is clean or max rounds are reached.

## Parameters

Parse the command arguments with strict positional rules:

- Format: `/pr-iterate [max_rounds] [auto]`
- **All parameters are optional and positional**
- **Parsing rules:**
  1. First argument (if numeric): `max_rounds` (default: 3)
  2. Remaining argument: `auto` flag (any value enables auto mode)

- `max_rounds` (optional, default: 3) — Maximum full iteration rounds
- `auto` (optional) — Skip user approval between rounds; abort on unrecoverable
  failures

**Examples:**

- `/pr-iterate` → max_rounds=3, auto=false
- `/pr-iterate 5` → max_rounds=5, auto=false
- `/pr-iterate auto` → max_rounds=3, auto=true
- `/pr-iterate 5 auto` → max_rounds=5, auto=true

## State

Track state across all rounds:

```
State = {
  round: 1,
  max_rounds: 3,
  auto: false,
  pr_number: null,
  pr_url: null,
  last_comment_id: null,
  changes_per_round: [],
  review_items_addressed: [],
  review_items_skipped: [],
  ci_fix_attempts: 0
}
```

## Instructions

### Phase 1: Pre-flight

1. Verify not on `main` or `master` branch. If so, abort with error.
2. Auto-detect PR number:
   ```
   gh pr view --json number,url -q '.number'
   gh pr view --json number,url -q '.url'
   ```
3. If no PR exists, abort: "No PR found for current branch. Create one first."
4. Show PR info and announce round number.
5. If auto mode, show warning: "Running in AUTO mode — will automatically fix
   issues and push without asking. Continue? (yes/no)"

### Phase 2: Local Quality (`make check`)

1. Run `make check` and capture output.
2. **If `make check` passes:** Skip to Phase 3.
3. **If `make check` fails:** Attempt automatic fixes: a. Run auto-fixers based
   on failure type:
   - Go lint: `golangci-lint run --fix ./...`
   - Go format: `gofmt -w .`
   - Python lint: `ruff check --fix pundora/`
   - Python format: `ruff format pundora/`
   - General formatting: `make format-fix` (if target exists) b. Re-run
     `make check`. c. **If still failing:** Spawn a fixer subagent.

4. **Fixer subagent** (if auto-fixers didn't resolve):

   Use the Agent tool with `subagent_type: general-purpose`:

   ```
   Prompt: "Read the following `make check` output and fix all failing
   checks. The failures are:

   <check_output>
   {make check output}
   </check_output>

   Instructions:
   - Read the failing files and understand the errors
   - Make minimal, targeted fixes
   - Follow the project conventions in CLAUDE.md
   - Do NOT fix unrelated code
   - After fixing, run `make check` to verify

   Report what you fixed."
   ```

5. **If still failing after subagent fix:**
   - In manual mode: Show failures, ask user what to do (fix/skip/abort)
   - In auto mode: Abort with failure summary

6. Run `make check` one final time to confirm clean state.

### Phase 3: Commit, Rebase, Push

1. Check for uncommitted changes: `git status --porcelain`
2. **If no changes:** Skip to Phase 4 (nothing to commit).
3. **If changes exist:** a. Stage changed files (use specific file paths, not
   `git add -A`). b. Create a conventional commit following the project's commit
   standards:
   - Format: `<type>(<scope>): <Summary in sentence case>`
   - Types: build, chore, ci, config, docs, feat, fix, perf, refactor, style,
     test
   - Scopes: api, auth, humor, infra, msgs, tools
   - Header max 72 chars, prefer 50
   - Consult `.commitlintrc.js` for up-to-date rules
   - Use `fix` type for review fixes, `refactor` for quality improvements c.
     Fetch and rebase:
   ```
   git fetch origin main
   git rebase origin/main
   ```
   d. **If rebase conflicts:**
   - Attempt auto-resolution for simple conflicts
   - In manual mode: Show conflicts, ask user for help
   - In auto mode: Abort rebase (`git rebase --abort`), push without rebase e.
     Push:
   ```
   git push --force-with-lease
   ```

### Phase 4: CI Checks

1. Wait for CI to complete:
   ```
   gh pr checks --watch --fail-fast
   ```
2. **If CI passes:** Proceed to Phase 5.
3. **If CI fails:** a. Fetch failure details:

   ```
   gh pr checks --json name,state,conclusion
   ```

   b. Increment `ci_fix_attempts` (max 2 per round). c. **For commitlint
   failures:**
   - Read the commit message: `git log -1 --format=%B`
   - Fix the commit message with `git commit --amend`
   - Push: `git push --force-with-lease`
   - Re-wait for CI d. **For test/build failures:**
   - Fetch workflow logs:
     ```
     gh run list --branch $(git branch --show-current) --limit 1 --json databaseId -q '.[0].databaseId'
     ```
     Then: `gh run view {run_id} --log-failed`
   - Spawn a fixer subagent with `subagent_type: general-purpose`:

     ```
     Prompt: "CI failed on this PR. Read the following CI log output and
     fix the failures:

     <ci_logs>
     {failed log output}
     </ci_logs>

     Instructions:
     - Identify the root cause of each failure
     - Read the relevant source files
     - Make minimal fixes
     - Run the failing tests locally to verify: {test commands}
     - Follow project conventions in CLAUDE.md

     Report what you fixed."
     ```

   - After fixes: stage, commit (`fix(<scope>): Fix CI failures`), push
   - Re-wait for CI e. **If CI still fails after 2 fix attempts:**
   - In manual mode: Show failures, ask user (fix/skip/abort)
   - In auto mode: Abort with CI failure summary

### Phase 5: Claude Bot Review Analysis

1. Fetch the latest `claude[bot]` review comment:
   ```
   gh api repos/clamshell-ai/humor/issues/{pr_number}/comments \
     --jq '[.[] | select(.user.login=="claude[bot]")] | last'
   ```
2. Extract the comment `id` and `body`.
3. **If no comment exists or comment `id` == `last_comment_id`:**
   - Poll every 30 seconds for up to 3 minutes for a new comment.
   - If still no new comment: Skip to Phase 7 (exit round as success).
4. Update `last_comment_id` with the new comment's ID.
5. Spawn an analysis subagent with `subagent_type: general-purpose`:

   ```
   Prompt: "Analyze the following Claude bot code review comment for PR #{pr_number}.
   Your job is to verify the accuracy of each finding and classify its true
   severity.

   <review_comment>
   {comment body}
   </review_comment>

   Instructions:
   1. For EACH finding in the review:
      a. Read the actual source file and line referenced
      b. Verify whether the finding is accurate (the bot sometimes hallucinates
         or references stale code)
      c. Assess true severity — is a 'CRITICAL' really critical, or is it
         cosmetic?
      d. Classify as one of:
         - `must-fix`: Genuine bug, security issue, or correctness problem
         - `should-fix`: Real issue but lower impact (style, minor improvement)
         - `wont-fix`: False positive, already handled, or not worth changing
      e. Provide brief reasoning for each classification

   2. Output a structured summary:
   ```

   ## Must-Fix Items
   - [file:line] Description — Reasoning

   ## Should-Fix Items
   - [file:line] Description — Reasoning

   ## Won't-Fix Items
   - [file:line] Description — Reasoning (why skipping)

   ```

   Be skeptical of the review. Verify every claim against actual code.
   The bot frequently flags things that are not actually issues."
   ```

6. Parse the subagent's classifications.

### Phase 6: Apply Changes

1. **If no `must-fix` or `should-fix` items:** Skip to Phase 7 (exit as
   success).
2. **In manual mode:**
   - Display the classified items to the user
   - Ask: "I found X must-fix and Y should-fix items. Which should I address?
     (all/must-only/pick/skip)"
     - `all` — Fix all must-fix and should-fix items
     - `must-only` — Fix only must-fix items
     - `pick` — Let user select specific items
     - `skip` — Skip all, proceed to next round
3. **In auto mode:** Fix all `must-fix` and `should-fix` items automatically.
4. Spawn a fixer subagent with `subagent_type: general-purpose`:

   ```
   Prompt: "Fix the following review items in this codebase:

   <items_to_fix>
   {list of must-fix and/or should-fix items with file:line references}
   </items_to_fix>

   Instructions:
   - Read each referenced file before making changes
   - Make minimal, targeted fixes for each item
   - Follow project conventions (see CLAUDE.md)
   - Do NOT fix unrelated code or make style-only changes beyond what's listed
   - After all fixes, run `make check` to verify nothing is broken

   Report what you fixed and any items you could not address (with reasoning)."
   ```

5. Record fixed items in `review_items_addressed` and skipped items in
   `review_items_skipped`.
6. Run `make check` to verify fixes don't break anything.
   - If broken: attempt to fix, or revert problematic changes.

### Phase 7: Round Completion

1. Increment `round` counter.
2. Record changes made this round in `changes_per_round`.
3. **If changes were made in Phase 6 (review fixes applied):**
   - Loop back to Phase 2 to commit and push the new fixes.
4. **If no changes were needed:**
   - Exit with success summary.
5. **If `round > max_rounds`:**
   - Exit with summary noting max rounds reached.
6. **In manual mode (and changes were made):**
   - Show round summary
   - Ask: "Continue to round {next}? (yes/no)"
   - If no: Exit with summary.

## Exit Summary

After completing (success or max rounds), display:

```
## PR Iteration Summary

**PR:** #{pr_number} ({pr_url})
**Rounds completed:** {round - 1} of {max_rounds}
**Final CI status:** {passing/failing}

### Changes by Round
- Round 1: {description of changes}
- Round 2: {description of changes}

### Review Items
- Addressed: {count} items
  - {list with brief descriptions}
- Skipped: {count} items
  - {list with reasoning}

### Status
{One of:}
- All review items addressed, CI passing — PR is ready for merge!
- Max rounds reached — {N} items remaining (see above)
- Aborted — {reason}
```

## Subagent Strategy

| Phase              | Subagent? | Why                                    |
| ------------------ | --------- | -------------------------------------- |
| Pre-flight         | No        | Simple git/gh commands                 |
| make check fix     | Yes       | Needs AI to understand and fix errors  |
| Commit/rebase/push | No        | Mechanical git operations              |
| CI fix             | Yes       | Needs AI to read logs and fix code     |
| Review analysis    | Yes       | Core AI judgment — verify accuracy     |
| Apply changes      | Yes       | Needs AI to make targeted code changes |

## Important Notes

- Always use the Agent tool to spawn subagents for fix and analysis work
- Use `--force-with-lease` for all force pushes (never `--force`)
- Track `last_comment_id` to detect new reviews — don't re-analyze the same
  comment
- Poll for Claude bot comments (30s intervals, 3min max) since the bot may take
  time after CI completes
- In manual mode, always ask for user approval before applying review fixes
- In auto mode, automatically fix all must-fix and should-fix items; abort on
  unrecoverable failures
- Follow commit conventions from `.commitlintrc.js` for all commits
- Do NOT invoke `/commit` as a nested skill — create commits directly following
  the same conventions inline
- If `make check` passes and CI passes and no new review comment exists, exit
  early — the PR is clean

## Example Flow

```
User: /pr-iterate 3

You: PR Iteration — Round 1 of 3
     PR: #618 (https://github.com/clamshell-ai/humor/pull/618)

     Phase 2: Running `make check`...
     ✅ make check passed

     Phase 3: No uncommitted changes — skipping commit/push.

     Phase 4: Checking CI status...
     ⏳ Waiting for CI...
     ✅ CI passed

     Phase 5: Checking for Claude bot review...
     Found new review comment (id: 12345).
     Analyzing review findings...

     [Spawns analysis subagent]

     Review analysis complete:
     - 1 must-fix: Race condition in pkg/sync/manager.go:89
     - 2 should-fix: Missing error context in hermelos/handler.go:45, 67
     - 3 won't-fix: Style preferences (not actual issues)

     Phase 6: I found 1 must-fix and 2 should-fix items.
     Which should I address? (all/must-only/pick/skip)

User: all

You: [Spawns fixer subagent]

     Fixed 3 items:
     - pkg/sync/manager.go:89 — Added mutex guard
     - hermelos/handler.go:45 — Wrapped error with context
     - hermelos/handler.go:67 — Wrapped error with context

     Running `make check`... ✅ passed

     Round 1 complete. Changes were made — starting round 2...

     Phase 2: Running `make check`... ✅ passed
     Phase 3: Committing and pushing fixes...
     Created commit: fix(msgs): Address review findings
     Pushed to origin.

     Phase 4: Waiting for CI... ✅ passed
     Phase 5: No new review comment after 3 minutes — skipping.

     ## PR Iteration Summary

     **PR:** #618
     **Rounds completed:** 2 of 3
     **Final CI status:** Passing ✅

     ### Review Items
     - Addressed: 3 items
     - Skipped: 3 items (style preferences, not actual issues)

     ### Status
     All review items addressed, CI passing — PR is ready for merge!
```

### Auto Mode Example

```
User: /pr-iterate 5 auto

You: Running in AUTO mode — will automatically fix issues and push. Continue? (yes/no)

User: yes

You: PR Iteration — Round 1 of 5
     PR: #618

     Phase 2: `make check`... ❌ failed (golangci-lint errors)
     Running auto-fixers... `golangci-lint run --fix`
     Re-running `make check`... ✅ passed

     Phase 3: Committing fixes...
     Created commit: fix(humor): Fix lint errors
     Rebasing on origin/main... ✅ clean rebase
     Pushing... ✅ pushed

     Phase 4: Waiting for CI... ✅ passed

     Phase 5: Found new review comment. Analyzing...
     - 2 must-fix items, 1 should-fix item, 4 won't-fix items
     Auto-fixing 3 items...

     [Spawns fixer subagent]

     Fixed 3 items. `make check`... ✅ passed.

     Round 2: Committing, pushing, waiting for CI... ✅
     No new review. Done!

     ## PR Iteration Summary
     **Rounds:** 2 of 5 | **CI:** Passing ✅
     **Addressed:** 5 items | **Skipped:** 4 items (false positives)
     All review items addressed — PR is ready for merge!
```
