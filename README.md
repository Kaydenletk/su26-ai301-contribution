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
**Status:** Phase IV Complete — **PR closed as a duplicate** (issue was already fixed upstream by [#8281](https://github.com/actualbudget/actual/pull/8281))

---

## Phase I: Issue Selection ✅ DONE

### Why I Chose This Issue

I chose issue #8134 "Payee page associated rules count includes completed schedules" because it sat squarely in my JavaScript/TypeScript comfort zone while pushing me into an unfamiliar area: how a local-first app derives counts from a shared rules table. My learning goal was to practice tracing a bug through an application's real data flow instead of patching the first layer where I saw the symptom.

I'm interested in this because:

- The reported behavior is concrete and testable — a number on screen is wrong — so I could prove the fix with assertions rather than opinion.
- The area (payees and rules) is a contained corner of `loot-core`, which made it realistic for a first contribution.
- I wanted to learn how Actual separates user-created rules from the internal rules a schedule auto-generates.
- The project is actively maintained, with a `CONTRIBUTING.md` and an `AGENTS.md` that document the setup and conventions.

### Problem Summary

The Payees page shows an inflated "associated rules" count: a payee with 10 completed tax schedules displays "10 rules," but clicking through to manage those rules shows zero. This matters because the count is the only signal users have about whether a payee has automation attached — an inflated number sends them hunting for rules that aren't theirs and erodes trust in the page. I chose it because the mismatch between the count and the actual list pointed at a specific, findable filtering bug rather than a vague UX complaint.

### Acceptance Criteria (what "fixed" looks like)

- A payee's rule count reflects only user-created rules.
- Rules auto-generated by schedules — active **and** completed — are excluded from the count.
- The count matches the number of rules actually listed on the manage-rules screen.
- Existing payee tests continue to pass.

### Files / Modules Likely Involved (identified before coding)

- `packages/loot-core/src/server/payees/app.ts` — suspected the count is derived server-side here.
- `packages/desktop-client/src/components/payees/` — where the count is rendered (my initial, incorrect hypothesis).

**Setup:** Created the GitHub account, forked `actualbudget/actual`, cloned it locally, and created this Contribution README.

## Phase II: Reproduce & Plan ✅ DONE

### Environment Setup

**Setup approach:** followed the repository's own `README.md` / `AGENTS.md` instructions rather than a dev container — Actual documents a plain `yarn install` + `yarn start` flow, and the app runs locally with SQLite so no external services are needed.

Cloned the repository, checked out the branch `fix-issue-8134`, and ran `yarn install` followed by `yarn start`. The local environment came up without major dependency issues; the main friction was simply the size of the monorepo install on first run.

**Working branch:** `fix-issue-8134` in my fork, named after the issue number.

### Steps to Reproduce

1. Start the app locally with `yarn start` and open the demo budget.
2. Navigate to **Schedules** and create a new schedule, assigning it to a payee (e.g., "Dominion Power").
3. Go to the **Payees** page and confirm the associated rules count for that payee increased by 1.
4. Return to **Schedules** and mark the newly created schedule as **Completed**.
5. Go back to the **Payees** page and read the count for that payee.
6. **Expected:** The count ignores the completed schedule's auto-generated rule and drops back down.
7. **Actual:** The count still includes the completed schedule; clicking through to manage rules shows no matching user rules.

### Solution Plan (UMPIRE)

**Understand:** The Payees page reports a rule count that does not match the rules a user can actually see or edit. **Root cause hypothesis:** schedules auto-create a rule row in the same `rules` table as user-created rules, and the count is derived without distinguishing the two — so every schedule silently inflates the number.

**Match:** I searched for existing code that already separates schedule-owned rules from user rules and found `getAllRuleIdsFromSchedules()` — a helper that returns exactly the set of rule IDs a schedule owns. This is genuinely analogous: it is the same distinction the count needs, already solved elsewhere in the codebase.

**Plan:**

1. Locate where the count is computed. My first assumption was the React component in `packages/desktop-client/src/components/payees/`, but tracing the data flow (frontend → API call → server function → in-memory rules) showed the count is produced server-side by `getPayeeRuleCounts()` in `packages/loot-core/src/server/payees/app.ts`.
2. Build a `Set` of schedule-owned rule IDs via the existing `getAllRuleIdsFromSchedules()` helper and skip those IDs while counting.
3. Add tests in `packages/loot-core/src/server/payees/app.test.ts` covering completed-schedule rules, active-schedule rules, and standalone user rules.

**Review:** Self-reviewed against the repository's `AGENTS.md` and commit conventions (`[AI]` prefix, release note expectations) before opening the PR.

**Evaluate:** Reproducing steps 1–5 above should show the count drop when a schedule is completed, and the count should equal the number of rules listed on the manage-rules screen. All existing payee tests must still pass.

### Change in Approach (documented for consistency)

My Phase II plan initially proposed a client-side `.filter()` in the Payee row component. Tracing the data flow proved that wrong — the count never reaches the client as a list, only as a number — so the implemented fix in Phase III is **server-side** in `getPayeeRuleCounts()`. This is the one deliberate deviation between plan and final PR.

## Phase III: Build ✅ DONE

**Issue:** [actualbudget/actual#8134](https://github.com/actualbudget/actual/issues/8134)

### Problem

The Payees page shows inflated rule counts. A payee with 10 completed tax schedules shows "10 rules" — but clicking to manage rules shows 0. Each schedule auto-creates a rule in the database, and those rules were being counted alongside user-created rules.

### Root Cause

`getPayeeRuleCounts()` in `packages/loot-core/src/server/payees/app.ts` iterates all rules with no filtering. Schedule-owned rules (internal, auto-created) and user-created rules live in the same table and were counted the same way.

### Fix

Used the existing `getAllRuleIdsFromSchedules()` helper to build a `Set` of schedule-owned rule IDs, then skip them during counting:

```js
const scheduleRuleIds = new Set(await rules.getAllRuleIdsFromSchedules(""));

rules.iterateIds(rules.getRules(), "payee", (rule, id) => {
  const ruleId = rule.getId();
  if (ruleId != null && scheduleRuleIds.has(ruleId)) return;
  // ... count only user-created rules
});
```

### Implementation Progress

Files modified (branch `fix-issue-8134`):

- `packages/loot-core/src/server/payees/app.ts` (+8 / −0) — filtered schedule-owned rules out of `getPayeeRuleCounts()`
- `packages/loot-core/src/server/payees/app.test.ts` (+99 / −0) — added 3 tests covering the fix

Key commit:

- `e773b49f2` — [AI] Fix payee rule count excluding schedule-owned rules

### Challenges Faced

The main obstacle was that my planned fix was in the wrong layer. I set out to add a `.filter()` in the React Payee row component, but the component only ever receives a finished number — there is no array to filter. **Resolved** by tracing the data flow end to end (frontend → API call → server function → in-memory rules), which located the real defect in `getPayeeRuleCounts()` server-side. A second, smaller obstacle: TypeScript flagged a `string | undefined` mismatch on `rule.getId()`, which I resolved with an explicit `ruleId != null` guard — without it the `Set` lookup would have silently failed at runtime.

### Testing

Both manual and automated:

- **Manual:** reproduced steps 1–5 from Phase II in the local app — created a schedule for a payee, watched the count increment, marked the schedule completed, and confirmed the count dropped back down after the fix.
- **Automated:** 3 tests added in `packages/loot-core/src/server/payees/app.test.ts`, all passing:
  - Completed schedule rules are excluded from the count
  - Standalone user-created rules are still counted correctly
  - Active schedule rules are also excluded

The existing payee test suite continued to pass with no new failures, and CI on the PR was green.

### What I Learned

- **Trace data flow before writing code** — the bug was server-side, not UI-side as I initially assumed.
- **Reuse existing helpers before writing new code** — `getAllRuleIdsFromSchedules()` already encoded exactly the distinction I needed.
- **TypeScript caught a `string | undefined` mismatch** that would have been a silent runtime bug.

## Phase IV: Submit & Iterate ✅ DONE

### Pull Request

- **PR Link:** [actualbudget/actual#8328](https://github.com/actualbudget/actual/pull/8328)
- **Summary:** Filters schedule-owned rules out of `getPayeeRuleCounts()` so the Payees page reports only user-created rules instead of an inflated count that includes rules auto-generated by active and completed schedules.
- **Status:** **Closed — duplicate.** The issue had already been fixed and merged upstream by [#8281](https://github.com/actualbudget/actual/pull/8281) before I opened my PR.

The PR was opened (not a draft) against the upstream `master` branch and referenced the issue with a close keyword.

### Before / After Evidence

Reproducing Phase II steps 1–5 against a payee with one completed schedule and no user-created rules:

```
Before fix:  Payees page rule count = 1   →  Manage rules list shows 0 rules   (mismatch)
After fix:   Payees page rule count = 0   →  Manage rules list shows 0 rules   (matches)

Test output (packages/loot-core/src/server/payees/app.test.ts):
  ✓ completed schedule rules are excluded from the count
  ✓ standalone rules are still counted correctly
  ✓ active schedule rules are also excluded
```

### Acceptance Checklist

- [x] Tests added for new/changed behavior (3 tests in `app.test.ts`)
- [x] All tests passing (payee suite green, CI green on the PR)
- [x] Follows project style guide (`[AI]` commit prefix per `AGENTS.md`)
- [x] No breaking changes introduced (count logic only; no schema or API change)
- [ ] Documentation updated — not applicable (no user-facing docs affected)

### Maintainer Feedback Log

| Date       | Feedback                                                                                                                                                                                                                                          | My Response / Commit                                                                                                                                                                                                                                                                                                           |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 2026-06-25 | Maintainer **@Juulz** responded to my `Fixes #8134` reference: _"It's already fixed and merged."_ Issue #8134 had been resolved upstream by PR #8281, merged **2026-06-22 15:31 UTC** — 2.5 days before I opened mine at 01:10 UTC on 2026-06-25. | I pulled the latest `master`, read commit `208283a51` from #8281, and confirmed it fixed the same root cause in the same function (`getPayeeRuleCounts()`) using the same helper. I closed my PR **17 minutes after opening it** with an explanatory comment, rather than leaving a duplicate open for a maintainer to triage. |
| 2026-06-25 | Automated review (**CodeRabbit**) requested changes: my test setup in `app.test.ts` (lines 12–17) relied on `loadRules()` registration rather than the project's established fixture pattern.                                                     | Not addressed on this PR — it was closed as a duplicate before I could iterate. I carried the lesson forward: in Contribution 2 I matched the neighbouring test files' setup conventions from the start (commit `dd664aa1e`).                                                                                                  |

**Reviewer surfacing:** The PR referenced the issue, which brought maintainer **@Juulz** into the conversation within 17 minutes, and the repository's fork-PR welcome bot auto-routed it for review.

### Learnings & Reflections

**Technical gains:** I learned to trace a bug through the real data flow rather than trusting my first guess. My plan was to filter at the React component level; following the path (frontend → API call → server function → in-memory rules) proved the component only receives a finished number, and the defect lived server-side in `getPayeeRuleCounts()`. I also learned to search for an existing helper before writing new logic — `getAllRuleIdsFromSchedules()` already encoded the exact user-rule vs. schedule-rule distinction I needed — and TypeScript's `string | undefined` warning on `rule.getId()` caught a `Set` lookup that would have silently failed at runtime.

**What I'd do differently:** Before writing a single line, I would verify the bug still exists on the latest `master`. I spent an entire cycle fixing something that had already been merged 2.5 days earlier. Concretely: I would `git pull upstream master`, search closed/merged PRs for the issue number, and re-run the reproduction steps against current code — a two-minute check that would have redirected me to a live issue. I would also fill in the PR description at open time; mine was effectively empty, which gives a reviewer no reason to engage even when the code is sound.

**Teachable insight for future cohorts:** **An open issue is not the same as an unfixed issue.** Maintainers frequently merge a fix without the issue auto-closing, so the issue tracker lags behind `master` — sometimes by days. Before claiming any issue, do three checks: (1) search the repo's merged PRs for the issue number, (2) read the current code on the default branch to confirm the bug is still there, and (3) reproduce it locally on fresh `master`. If you cannot reproduce it, it is already fixed — pick another issue. My PR was closed 17 minutes after opening because I skipped this. The check costs two minutes; skipping it cost a full contribution cycle.
