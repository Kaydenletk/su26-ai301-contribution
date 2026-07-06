# Contribution 2: Make budget template notifications translatable

**Contribution Number:** 2
**Student:** Kayden Le
**Issue:** [actualbudget/actual#8276](https://github.com/actualbudget/actual/issues/8276)]
**Status:** Phase I

---

## Why I Chose This Issue
I chose issue #8276 "[AI] Fix daily auto-post schedules skipping days" because it aligns perfectly with my TypeScript skills and my goal to improve my debugging and unit testing abilities. The issue tackles a specific logic bug involving date handling, which is a critical and highly transferable skill in backend engineering.

I'm interested in this because:

It allows me to continue contributing to the Actual Budget codebase, deepening my understanding of its loot-core package.

The problem area (specifically schedules.ts and getHasTransactionsQuery) is well-contained, making it an excellent opportunity to write robust integration and unit tests.

I want to learn how to properly handle date-bounding and filter constraints to prevent out-of-order data from breaking core logic.

The issue has clear, testable acceptance criteria: daily schedules must post sequentially without skipping days.

From reading the issue and debugging the code, I understand the current problem is that later, out-of-order transactions cause earlier schedule occurrences to falsely appear as paid because the query lacks a strict upper bound. My contribution will fix this by adding an $lte: next_date constraint to the transaction date filter, ensuring reliable daily catch-ups
## Reproduction Process

### Environment Setup
Cloned the monorepo and ran `yarn install` (Node 22, Yarn 4) — had it running in
about 30 minutes. No dev container needed; `loot-core` is platform-agnostic, so
the bug reproduces headless through the Vitest suite. Only snag was that after
branching off freshly-fetched upstream commits, a second `yarn install` was
required to relink workspaces and rebuild native deps (`argon2`, `bcrypt`).

Working branch: https://github.com/Kaydenletk/actual/tree/fix-daily-schedule-gaps-8276

### Steps to Reproduce
1. Create a **daily** schedule with "Post transactions automatically" enabled,
   starting a few days in the past (e.g. 2016-12-28)
2. Ensure a transaction already exists for a **later** day in that range
   (2016-12-30) — from an earlier run, a manual entry, or an out-of-order sync
3. Trigger the auto-post catch-up (runs on sync / app load)
4. **Expected:** A transaction is posted for every missed day — 12-28, 12-29,
   12-30, 12-31, 01-01
5. **Actual:** Only 12-30, 12-31, 01-01 are posted; **12-28 and 12-29 are
   silently skipped**, leaving the gaps the issue reports

### Solution Plan
**Understand:** `getHasTransactionsQuery` decides whether a schedule occurrence
is already "paid," but filters transactions with only a lower date bound
(`date >= next_date`) and no upper bound. For a daily schedule that asks "is
there ANY transaction on or after this day?" — so a single later transaction
makes every earlier unposted occurrence look already-paid, and the catch-up
loop advances past those days without posting.

**Match:** The forecast path already dedups per-occurrence with an upper bound
in `isScheduleOccurrencePosted` at `packages/loot-core/src/shared/schedules.ts`
line 105 (`tx.date >= matchStart && tx.date <= occurrenceDate`). The query path
never got the same bound.

**Plan:**
1. Add an upper bound `date <= next_date` to `getHasTransactionsQuery` in
   `packages/loot-core/src/shared/schedules.ts` line 124, using the array
   condition form `date: [{ $gte }, { $lte }]` — a single `{ $gte, $lte }`
   object silently drops the second operator because the AQL compiler keeps
   only the first key.
2. Add 1 integration test in
   `packages/loot-core/src/server/schedules/app.test.ts` and 2 unit tests in
   `packages/loot-core/src/shared/schedules.test.ts`.
3. Run the full loot-core test suite to confirm no regressions.

**Review:** Will self-review against the repo's `AGENTS.md` and commit
conventions (`[AI]` title prefix, blank PR template, release note added) before
opening the PR.

**Evaluate:** Manual test reproducing steps 1-3 above should now post a
transaction for every day with no gaps. All 3 new tests should fail without the
fix and pass with it; all existing tests should continue to pass.****
---
---

# Contribution 1: Payee page associated rules count includes completed schedules

**Contribution Number:** 1
**Student:** Kayden Le
**Issue:** [actualbudget/actual#8134](https://github.com/actualbudget/actual/issues/8134)
**Pull Request:** [actualbudget/actual#8328](https://github.com/actualbudget/actual/pull/8328)
**Status:** Phase IV Complete — Awaiting review

## Phase IV: ✅ DONE

Pull Request
PR Link: [https://github.com/actualbudget/actual/pull/8259](https://github.com/actualbudget/actual/pull/8328)

What I Contributed
Issue: #8134 — Payee page associated rules count includes completed schedules

Fixed a bug where the Payees page displayed inflated rule counts by including rules auto-generated by schedules (active and completed). Users would see a non-zero count, click to manage rules, and find nothing — because those rules belonged to schedules, not user-created rules.

Files changed:

packages/loot-core/src/server/payees/app.ts — filtered out schedule-owned rules from getPayeeRuleCounts()
packages/loot-core/src/server/payees/app.test.ts — added 3 tests covering the fix
Feedback / Next Steps
No feedback received yet. PR is open and awaiting maintainer review.

Status
Awaiting Review

Learnings & Reflections
Biggest lesson: The fix I planned (filtering at the React component level) was wrong. Tracing the actual data flow — frontend → API call → server function → in-memory rules — showed the bug lived one layer deeper than expected. Always follow the data before writing a single line of code.

## Phase III: ✅ DONE

Issue: https://github.com/actualbudget/actual/issues/8134

Problem
Payees page shows inflated rule counts. A payee with 10 completed tax schedules shows "10 rules" — but clicking to manage rules shows 0. Each schedule auto-creates a rule in the database, and those rules were being counted alongside user-created rules.

Reproduction
Create a schedule assigned to a payee
Observe rule count increases by 1 on Payees page
Mark schedule as Completed
Actual: count stays the same
Expected: count drops back down
Root Cause
getPayeeRuleCounts() in packages/loot-core/src/server/payees/app.ts iterates all rules with no filtering. Schedule-owned rules (internal, auto-created) and user-created rules live in the same table and were counted the same way.

Fix
Used the existing getAllRuleIdsFromSchedules() helper to build a Set of schedule-owned rule IDs, then skip them during counting:

const scheduleRuleIds = new Set(await rules.getAllRuleIdsFromSchedules(''));

rules.iterateIds(rules.getRules(), 'payee', (rule, id) => {
const ruleId = rule.getId();
if (ruleId != null && scheduleRuleIds.has(ruleId)) return;
// ... count only user-created rules
});
Tests
3 tests added in app.test.ts — all pass:

Completed schedule rules excluded from count
Standalone rules still counted correctly
Active schedule rules also excluded
What I Learned
Trace data flow before writing code — bug was server-side, not UI-side as initially assumed
Reuse existing helpers before writing new code
TypeScript caught a string | undefined type mismatch that would have been a silent runtime bug
Commit: [AI] Fix payee rule count excluding schedule-owned rules

## Phase II: Reproduce & Plan ✅ DONE

### Reproduction Process

**Environment Setup:** Cloned the repository, checked out branch `fix-issue-8134`, and ran `yarn install` followed by `yarn start`. The local environment ran successfully without major dependency issues.

**Working Branch:** https://github.com/actualbudget/actual/issues/8134

**Steps to Reproduce:**

1. Navigate to Schedules in the local Actual Budget app.
2. Create a new schedule and assign it to a payee (e.g., "Dominion Power").
3. Verify on the Payees page that the rule count increases by 1.
4. Go back to Schedules and mark the newly created schedule as "Completed".
5. Return to the Payees page.
6. **Actual Result:** The associated rules count incorrectly includes the completed schedule.
7. **Expected Result:** The count should ignore completed schedules and drop back down.

### Solution Approach (Implementation Plan)

- **Understand:** The UI component displaying the "associated rules" count for a payee is pulling the total length of an array without checking if the associated schedules have a status of "completed".
- **Match:** I will look for existing utility functions in the codebase that filter active schedules versus completed schedules.
- **Plan:** I will locate the specific React component rendering the Payee row (likely in `packages/desktop-client/src/components/payees/`). Before the component calculates the `.length` of the rules, I will add a `.filter()` condition to exclude any schedules marked as completed.
- **Evaluate:** I will manually test this by creating and completing a schedule in my local environment. The UI counter must drop immediately upon completion without breaking other Payee page features.
