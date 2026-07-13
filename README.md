# Contribution 2: Fix daily auto-post schedules skipping days

**Contribution Number:** 2
**Student:** Kayden Le
**Issue:** [actualbudget/actual#8276](https://github.com/actualbudget/actual/issues/8276)
**Pull Request:** [actualbudget/actual#8418](https://github.com/actualbudget/actual/pull/8418)
**Status:** Phase IV Complete — Awaiting review

---

## Phase I: Issue Selection ✅ DONE

### Why I Chose This Issue

I chose issue #8276 "[AI] Fix daily auto-post schedules skipping days" because it aligns perfectly with my TypeScript skills and my goal to improve my debugging and unit testing abilities. The issue tackles a specific logic bug involving date handling, which is a critical and highly transferable skill in backend engineering.

I'm interested in this because:

- It allows me to continue contributing to the Actual Budget codebase, deepening my understanding of its `loot-core` package.
- The problem area (specifically `schedules.ts` and `getHasTransactionsQuery`) is well-contained, making it an excellent opportunity to write robust integration and unit tests.
- I want to learn how to properly handle date-bounding and filter constraints to prevent out-of-order data from breaking core logic.
- The issue has clear, testable acceptance criteria: daily schedules must post sequentially without skipping days.

### Problem Summary

Daily schedules set to post automatically leave gaps instead of posting one transaction per day. This matters because users rely on these schedules to track recurring cash flow — missed days silently understate spending and break budget forecasts. I chose it because a prior fix (#7299) targeted the same behavior but didn't fully resolve it, which told me the real root cause was still open and worth tracing. From reading the issue and debugging the code, the underlying problem is that later, out-of-order transactions cause earlier schedule occurrences to falsely appear as paid, because the "already posted?" query lacks a strict upper date bound.

### Acceptance Criteria (what "fixed" looks like)

- A daily auto-post schedule posts exactly one transaction for every day in its catch-up range.
- A pre-existing transaction on a later day never suppresses posting for earlier days.
- No duplicate transactions are created for days that already have one.
- All existing schedule tests continue to pass.

**Setup:** Forked and cloned the repo, created the branch `fix-daily-schedule-gaps-8276`, and created this Contribution README.

## Phase II: Reproduce & Plan ✅ DONE

### Environment Setup

Cloned the monorepo and ran `yarn install` (Node 22, Yarn 4) — had it running in about 30 minutes. **Setup approach:** followed the repo's `AGENTS.md` / `README.md` instructions rather than a dev container, since `loot-core` is platform-agnostic and the bug reproduces headless through the Vitest suite.

**Real challenges encountered and how I resolved them:**

- After branching off freshly-fetched upstream commits, the build failed with `Cannot find package '@actual-app/vite-plugin-peggy'`. **Fix:** re-ran `yarn install` from the repo root to relink the workspaces.
- The sync-server tests then failed with `Cannot find module 'argon2'` / `bcrypt`. **Fix:** these are native modules — the second `yarn install` rebuilt them against the local Node 22 toolchain.

**Working branch:** [fix-daily-schedule-gaps-8276](https://github.com/Kaydenletk/actual/tree/fix-daily-schedule-gaps-8276) (named after the issue).

### Steps to Reproduce

1. Create a **daily** schedule with "Post transactions automatically" enabled, starting a few days in the past (e.g. `2016-12-28`).
2. Ensure a transaction already exists for a **later** day in that range (`2016-12-30`) — from an earlier run, a manual entry, or an out-of-order sync.
3. Trigger the auto-post catch-up (runs on sync / app load).
4. **Expected:** A transaction is posted for every missed day — `12-28, 12-29, 12-30, 12-31, 01-01`.
5. **Actual:** Only `12-30, 12-31, 01-01` are posted; **`12-28` and `12-29` are silently skipped**, leaving the gaps the issue reports.

**Evidence (test harness output):**

```
Before fix: ["2016-12-30","2016-12-31","2017-01-01"]   ← 12-28, 12-29 skipped
```

### Solution Plan (UMPIRE)

**Understand:** `getHasTransactionsQuery` decides whether a schedule occurrence is already "paid," but filters transactions with only a lower date bound (`date >= next_date`) and no upper bound. For a daily schedule that asks "is there ANY transaction on or after this day?" — so a single later transaction makes every earlier unposted occurrence look already-paid, and the catch-up loop advances past those days without posting. **Root cause:** the missing upper bound on the date filter, in `packages/loot-core/src/shared/schedules.ts`.

**Match:** The forecast path already dedups per-occurrence _with_ an upper bound in `isScheduleOccurrencePosted` at `packages/loot-core/src/shared/schedules.ts` line 105 (`tx.date >= matchStart && tx.date <= occurrenceDate`). The query path never got the same bound — a genuinely analogous example already in the codebase.

**Plan:**

1. Add an upper bound `date <= next_date` to `getHasTransactionsQuery` in `packages/loot-core/src/shared/schedules.ts` (line 124), using the array condition form `date: [{ $gte }, { $lte }]` — a single `{ $gte, $lte }` object silently drops the second operator because the AQL compiler keeps only the first key.
2. Add 1 integration test in `packages/loot-core/src/server/schedules/app.test.ts` and 2 unit tests in `packages/loot-core/src/shared/schedules.test.ts`.
3. Run the full `loot-core` test suite to confirm no regressions.

**Review:** Self-reviewed against the repo's `AGENTS.md` and commit conventions (`[AI]` title prefix, blank PR template for the human, release note added) before opening the PR.

**Evaluate:** Manual test reproducing steps 1–3 above now posts a transaction for every day with no gaps. All 3 new tests fail without the fix and pass with it; all existing tests continue to pass.

## Phase III: Build ✅ DONE

**Issue:** [actualbudget/actual#8276](https://github.com/actualbudget/actual/issues/8276)

### Problem

Daily schedules with "post automatically" enabled post transactions irregularly, leaving gaps instead of one transaction per day. A prior fix (#7299, disabling the 2-day lookback) did not fully resolve it.

### Root Cause

`getHasTransactionsQuery()` in `packages/loot-core/src/shared/schedules.ts` decides whether an occurrence is already "paid," but filtered transactions with only a lower date bound (`date >= next_date`) and no upper bound. For a daily schedule that asks "is there ANY transaction on or after this day?" — so a single later transaction makes every earlier unposted occurrence look already-paid, and the catch-up loop advances past those days without posting.

### Fix

Added an upper bound so `hasTrans` reflects only the current occurrence. Because the AQL compiler keeps only the first key of a condition object, the range is expressed as an array (implicitly AND-ed):

```ts
date: [
  { $gte: getScheduleOccurrenceMatchStartDate(schedule, schedule.next_date) },
  { $lte: schedule.next_date },
],
```

### Implementation Progress

Files modified (branch `fix-daily-schedule-gaps-8276`):

- `packages/loot-core/src/shared/schedules.ts` — added the `$lte: next_date` upper bound to `getHasTransactionsQuery`
- `packages/loot-core/src/server/schedules/app.test.ts` — integration test for daily catch-up with a later transaction
- `packages/loot-core/src/shared/schedules.test.ts` — 2 unit tests asserting the query's `[matchStart, next_date]` range
- `upcoming-release-notes/fix-daily-schedule-skipped-days.md` — user-facing changelog entry

Key commits:

- `dd664aa1e` — [AI] Fix daily auto-post schedules skipping days (#8276)
- `d95fe3d35` — [AI] Reword schedule release note to plain language

### Challenges Faced

The one-line conceptual fix (`$lte: next_date`) did not work at first: the query still matched later transactions. The obstacle was that a single `{ $gte, $lte }` object silently dropped the second operator. **Resolved** by reading the AQL compiler (`compiler.ts`) and discovering it keeps only the first key of a condition object; switching to the array form `date: [{ $gte }, { $lte }]` (which the compiler AND-s) fixed it.

### Testing

Both manual and automated:

- **Manual:** ran the catch-up in the test harness and inspected the posted dates before and after the fix (see before/after evidence in Phase IV).
- **Automated:** 3 tests added — all fail without the fix and pass with it (verified by stashing the change):
  - Daily catch-up posts every day and skips none when a later transaction exists (`app.test.ts`)
  - `getHasTransactionsQuery` bounds an auto-posted schedule to its exact occurrence date (`schedules.test.ts`)
  - `getHasTransactionsQuery` caps the manual 2-day lookback at the occurrence date (`schedules.test.ts`)

Existing suites still green: `loot-core` 979 pass, schedules 75 pass, api 20 pass. `yarn typecheck` and `yarn lint` clean.

### What I Learned

- **Follow the data** — the visible symptom (irregular posting) lived in a query's missing date bound, not the catch-up loop itself.
- **Verify framework semantics** — the AQL compiler silently dropped an operator that looked correct in JavaScript.
- **Reuse existing patterns** — an existing helper (`isScheduleOccurrencePosted`) already applied the correct upper bound; the query path just never got it.

## Phase IV: Submit & Iterate ✅ DONE

### Pull Request

- **PR Link:** [actualbudget/actual#8418](https://github.com/actualbudget/actual/pull/8418)
- **Summary:** Adds an upper date bound to `getHasTransactionsQuery` so daily auto-post schedules post a transaction for every day instead of skipping days masked by a later transaction.
- **Status:** Awaiting review

The PR is open (not a draft) against the upstream `master` branch, uses the project's PR template (Description / Related issue / Testing / Checklist), and closes the issue with `Closes #8276`.

### Before / After Evidence

Test-harness output for a daily schedule starting `2016-12-28` with a pre-existing transaction on `2016-12-30`:

```
Before fix: ["2016-12-30","2016-12-31","2017-01-01"]                              ← 12-28, 12-29 skipped
After fix:  ["2016-12-28","2016-12-29","2016-12-30","2016-12-31","2017-01-01"]    ← every day posted
```

### Acceptance Checklist (in PR)

- [x] Tests added (1 integration + 2 unit)
- [x] All tests passing (`loot-core` 979 pass)
- [x] Follows style guide (`yarn lint` + `yarn typecheck` clean)
- [x] No breaking changes (existing schedule/forecast suites unchanged)
- [x] Release note added

### Maintainer Feedback Log

| Date       | Feedback                                                                                                                                                                                                                                    | My Response / Commit |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| 2026-07-06 | No maintainer feedback received yet. PR opened, `[WIP]` prefix and WIP label removed, "ready for review" label applied, all CI checks green (42/45, 3 skipped). CodeRabbit (automated review) approved. Awaiting a human maintainer review. | —                    |

The reviewer was surfaced via a `🤖` comment on the PR asking a maintainer to drop the WIP label and review; the repo's welcome bot also auto-routed the PR for review.

### Learnings & Reflections

**Technical gains:** I learned how a query DSL compiles down — specifically that the AQL compiler reads only the first key of a condition object, so range filters must be expressed as an array of single-operator objects. I also learned to locate an analogous, already-correct pattern in a codebase (`isScheduleOccurrencePosted`) and mirror it, rather than inventing a new approach.

**What I'd do differently:** I'd check how the framework interprets a filter _before_ assuming my JavaScript object maps 1:1 to SQL — I lost time debugging a fix that was conceptually right but silently mis-compiled. Next time I'll write the smallest possible failing test against the query layer first, then build the fix on top of it.

**Teachable insight for future cohorts:** When a "one-line fix" doesn't work, suspect the layer _below_ your code (the compiler / ORM / serializer), not your logic. A change that reads correctly in the source language can be silently altered by the tool that translates it — always verify the generated output, not just the input.

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
