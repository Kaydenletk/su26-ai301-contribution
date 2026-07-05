# Contribution 2: Make budget template notifications translatable

**Contribution Number:** 2
**Student:** Kayden Le
**Issue:** [actualbudget/actual#8413](https://github.com/actualbudget/actual/issues/8413)
**Pull Request:** [actualbudget/actual#8417](https://github.com/actualbudget/actual/pull/8417)
**Status:** Phase IV Complete — Awaiting review

---

## Why I Chose This Issue

I chose issue #8413, "Notifications for some budget template actions are always shown in English (not marked as translatable)," because it's a direct continuation of my Contribution 1 work in the same codebase (Actual Budget, a TypeScript/React local-first budgeting app) — this time on the internationalization (i18n) layer rather than the data-counting layer. It's labeled `good first issue`, `help wanted`, and `goal templates`, so it's an explicitly welcomed, well-scoped task rather than a stale or contentious one.

I also chose it deliberately because it looked _simpler than it actually is_. On the surface it reads as "wrap three strings in `t()`" — the kind of one-line fix that a first-time contributor grabs and gets wrong. I wanted to find out whether that surface reading held up, and it didn't: the strings live in `loot-core`, a package that a very recent architectural decision (PR #8027) had **deliberately stripped of all i18next dependencies**. That constraint is the whole reason this bug exists and can't be fixed the obvious way. Discovering that early is what separated this from a naive patch.

Before claiming, I checked the issue comments: another contributor (@Andrei-hub11) had said "I'd like to work on this," and a maintainer (@MatissJanis) replied "Feel free to send a PR — we do not assign issues." So the issue is open to whoever ships a working PR first, not assigned. I proceeded on that basis.

---

## Understanding the Issue

### Problem Description

The notification messages shown when a user runs the **"Check templates," "Apply budget template,"** or **"Overwrite with budget template"** actions from the budget month header are always rendered in English, regardless of the app's configured language. The issue reporter (running the desktop Electron app on Linux with a non-English locale) also noted these strings do not appear on Weblate (the project's translation platform) at all — meaning translators never even get the chance to translate them.

The specific user-facing strings called out:

- `"All templates passed! 🎉"`
- `"Everything is up to date"`
- `"Successfully applied templates to N categories"`
- ...and, per the reporter, "probably some error messages I didn't trigger."

### Expected Behavior

These notification strings should be marked as translatable (extracted into the i18next catalog / `locale/en.json`, surfaced on Weblate) and rendered in the user's configured language, exactly like every other user-facing string in the app.

### Current Behavior

The strings are hardcoded English literals inside notification objects returned from server-side code in `loot-core`. They are sent to the client and displayed verbatim, never passing through any translation function.

### Affected Components

- `packages/loot-core/src/server/budget/goal-template.ts` (`processTemplate` — "Everything is up to date", "Successfully applied templates to N categories", "There were errors interpreting some templates:")
- `packages/loot-core/src/server/budget/template-notes.ts` (`checkTemplateNotes` — "All templates passed! 🎉", plus the same "errors interpreting" header)
- `packages/desktop-client/src/budget/mutations.ts` (`useBudgetActions` — **the actual translation boundary; this file is not mentioned in the issue at all, and is where the fix ultimately had to live**)
- `packages/loot-core/src/server/budget/cleanup-template.ts` (a _third_ `Notification` type definition I had to touch for the client-side type union to compile — discovered only via the type checker, not by reading the issue)

---

## Reproduction Process

### Environment Setup

Cloned my fork of the repository, created branch `fix-issue-8413` off `origin/master`, and prepared to run the app. Actual is a Yarn 4 workspace monorepo (`loot-core`, `desktop-client`, `api`, `sync-server`, `component-library`, `crdt`, and more) orchestrated with the **lage** task runner. Standard verification commands are `yarn typecheck`, `yarn lint`, and `yarn test`, all run from the repository root (running yarn inside a child workspace is blocked by a repo git hook).

### Steps to Reproduce

Per the issue:

1. Set the app's language to something other than English.
2. Go to the Budget page.
3. Define a template or budget automation for at least one category (e.g. add `#template 100` to a category's notes).
4. Open the three-dots menu in the month header.
5. Run **"Check templates," "Apply budget template," "Overwrite with budget template."**
6. **Actual:** the resulting toast notifications appear in English regardless of the selected language.
7. **Expected:** the notifications appear in the selected language.

### Reproduction Evidence

- **My finding (traced, not assumed):** I traced the exact data path that produces these notifications rather than assuming the strings could be wrapped in place. The chain is:
  `useBudgetActions()` (`mutations.ts`) → `send('budget/check-templates')` / `send('budget/apply-goal-template')` / `send('budget/overwrite-goal-template')` → server handler in `loot-core/src/server/budget/app.ts` → `runCheckTemplates()` / `applyTemplate()` / `overwriteTemplate()` in `goal-template.ts` → returns a `Notification` object with a hardcoded English `message`.
- On the client, `mutations.ts`'s `onSuccess` handler took that raw server `Notification` and dispatched it **verbatim** into Redux via `addNotification({ notification })` — no `t()` anywhere in between. That untouched passthrough is the exact bug.
- **Key structural finding:** the strings can't simply be wrapped in `t()` where they're defined, because `loot-core` has no access to i18next (see Solution Approach → the #8027 constraint). This is why the issue's naive framing ("mark them translatable") is more subtle than it looks.

---

## Solution Approach

### Analysis — the constraint that shapes everything (PR #8027)

Before writing any code, I researched how this codebase makes _server-generated_ notification messages translatable, since `loot-core/src/server/` runs in a background worker without React/i18next hooks available the usual way. The critical discovery:

**PR #8027, "[AI] Move i18n usage outside of loot-core" (a deliberate, recent architectural decision), did the exact opposite of the obvious fix.** It _removed_ `import { t } from 'i18next'` from several `loot-core` files and removed the `i18next` dependency from `loot-core`'s `package.json` entirely, with the stated rationale that _"loot-core should be platform-agnostic and free of i18n concerns."_

So the "obvious" fix — `import { t } from 'i18next'` in `goal-template.ts` and wrap the strings — is precisely what a maintainer just went out of their way to forbid, and the dependency isn't even present to import anymore. Any PR that did that would be correctly rejected. **The fix has to translate at the client boundary, not at the string's origin.**

### Investigative Depth

- I confirmed the existing working pattern for "translate a string in non-React code without a hook": `packages/desktop-client/src/notifications/notificationsSlice.ts` uses a **module-level** `import { t } from 'i18next'` (not the `useTranslation` hook) inside `addGenericErrorNotification`. This works because `i18next` is initialized as a singleton in `desktop-client` at app startup, so a plain `t()` import resolves anywhere in that package. That's the precedent for translating on the client side.
- I checked `packages/desktop-client/i18next-parser.config.js` and found its `input` array explicitly includes `'../loot-core/src/**/*.{js,jsx,ts,tsx}'` — i.e. the string-extraction tooling _does_ scan `loot-core` source. But the extractor can only pull out **literal** `t('...')` calls; it cannot follow a `t(variable)` call or a bare object literal `message: '...'`. This detail turned out to be decisive (see Challenges Faced).
- I confirmed the client consumption point precisely: `mutations.ts`'s `useBudgetActions` already had `const { t } = useTranslation();` in scope (used only by its `onError` branch), but the server-supplied `notification.message` flowed straight past it into `addNotification` untranslated.

### Match

The most directly analogous existing pattern is `notificationsSlice.ts:addGenericErrorNotification`, which translates a non-React-originated message via a module-level `t()`. My fix follows that same principle — translate at the point the message enters the client — but adapts it to the fact that the message text originates on the server and arrives as a runtime value.

### Edge cases identified proactively

- **Dynamic interpolation can't be extracted.** The string `` `Successfully applied templates to ${contexts.length} categories` `` is a template literal with a runtime value. i18next-parser cannot extract a key from an interpolated template literal, and translating a fully-formed English sentence client-side is impossible if the number is already baked in. **Fix:** the server now returns the _key_ form `'Successfully applied templates to {{count}} categories'` (a static literal) plus a separate `values: { count: contexts.length }` object, so the client can interpolate at translation time via `t(key, values)`.
- **`t(variable)` is invisible to the extractor.** A generic `t(notification.message, notification.values)` at the boundary translates correctly at runtime but extracts _nothing_ into `en.json` — so translators would still never see the strings, and the "not on Weblate" half of the bug would remain unfixed. **Fix:** an explicit `switch` on the known message with a literal `t('...')` call per case, so the parser has a static literal to extract for each one. (This is the standard i18next "the parser can't follow variables" workaround.)
- **The error-detail block mixes translatable UI text with untranslatable parser output.** `checkTemplateNotes`'s `errors[]` array (joined into the notification's `pre` field) interleaves genuinely translatable copy (e.g. `Schedule "X" does not exist`) with raw PEG-parser error dumps (`(e as Error).message`) that are not user-authored, English-only, and out of scope to i18n. Fully translating that block would require reshaping the whole server→client notification pipeline into structured error objects. **Decision:** I scoped the fix to the four notification strings the issue explicitly names, and deliberately left the raw parser-error `pre` block in English, documenting that choice rather than silently half-doing it. (I confirmed this scoping decision explicitly before implementing.)

### Proposed Solution

1. **Server (`goal-template.ts`, `template-notes.ts`):** keep returning English strings, but as **stable i18next keys** — static literals, with any dynamic value moved out of the string and into a new optional `values?: Record<string, unknown>` field on the `Notification` type. No i18next import added (respecting #8027).
2. **Client (`mutations.ts`):** at the `onSuccess` notification boundary, run the server message through a small `translateBudgetTemplateMessage(t, notification)` helper — a `switch` with one literal `t('...')` call per known message (so the extractor sees them), falling back to the raw message for anything unrecognized.
3. Regenerate the i18n catalog and confirm all four strings land in `locale/en.json`.

### Implementation Plan (as executed)

1. Add `values?: Record<string, unknown>` to the `Notification` type in `goal-template.ts`.
2. Change the dynamic success message to the key form + `values: { count }`.
3. Add the same `values?` field to `template-notes.ts`'s `Notification` type (its two strings are static, but the field keeps the shape consistent for the client union).
4. Add the same `values?` field to `cleanup-template.ts`'s `Notification` type — **not in the issue's scope**, but required because the `cleanup-goal-template` action routes through the _same_ `useBudgetActions` `onSuccess`, so its notification shape is part of the client-side type union; without it the whole union failed to typecheck. Found by the type checker, not by reading the issue.
5. Add the `translateBudgetTemplateMessage` helper + wire it into `onSuccess` in `mutations.ts`.
6. Test, lint, typecheck, regenerate i18n, add a release note.

### Review checklist

- [x] Does it keep `loot-core` free of i18next, per PR #8027? — Yes; no i18next import was added to any `loot-core` file. Translation happens entirely in `desktop-client`.
- [x] Do all four strings actually get extracted into `locale/en.json`? — Yes; verified by running `yarn generate:i18n` and grepping the output (see Testing Strategy). This is the half of the bug — "not on Weblate" — that a naive `t(variable)` fix would silently leave broken.
- [x] Is the dynamic count still correct and translatable? — Yes; `Successfully applied templates to {{count}} categories` interpolates `values.count` at translation time.
- [x] Is the error-`pre` scoping decision documented rather than silent? — Yes (see Edge cases).

---

## Testing Strategy

### Unit Tests

The change's testable logic (the message key + the `values` shape) lives in `loot-core`, so I extended the existing server test rather than writing a brittle React-hook test:

- `packages/loot-core/src/server/budget/goal-template.test.ts` — the existing test `"writes per-category budgets and returns a success notification"` already exercises the exact `processTemplate` path I changed. I added one assertion, `expect(result.values).toEqual({ count: 2 })`, to lock in the new server contract (message is now a key; the count lives in `values`). The pre-existing `expect(result.message).toMatch(/Successfully applied/)` still passes unchanged, because the key text still contains that substring.

All **194** tests in `packages/loot-core/src/server/budget/` pass after the change (8 test files).

### Why no `mutations.ts` unit test

`useBudgetActions` is a React Query mutation hook. There is **no** existing unit test for `mutations.ts` anywhere in the repo, and no e2e coverage of the check-templates flow — so there was no established harness to extend, and standing one up (mock QueryClient + Redux provider + i18next) would have been disproportionate scaffolding for a mechanical `t()`-at-the-boundary change. I flagged this explicitly rather than silently skipping it, and put the meaningful assertion where the logic actually is (the server contract, in `loot-core`).

### i18n extraction verification (the key check for _this_ issue)

Because the whole point of the issue is "these strings aren't translatable / aren't on Weblate," the single most important verification isn't a unit test — it's proving the strings now get extracted. I ran the project's own extraction command, `yarn generate:i18n` (which runs `i18next-parser`), and confirmed all four keys now appear in `packages/desktop-client/locale/en.json`:

```
"All templates passed! 🎉": "All templates passed! 🎉",
"Everything is up to date": "Everything is up to date",
"Successfully applied templates to {{count}} categories": "Successfully applied templates to {{count}} categories",
"There were errors interpreting some templates:": "There were errors interpreting some templates:",
```

Before my `switch`-with-literal-`t()`-calls approach, an earlier attempt using a generic `t(notification.message, notification.values)` translated correctly at runtime but extracted **zero** of these keys — proving out the "parser can't follow a variable" edge case in practice, not just in theory.

### Typecheck / Lint / Format

- `yarn typecheck` — clean across all 6 typechecked workspaces (this is what caught the missing `values` field on `cleanup-template.ts`'s `Notification` type: `Property 'values' does not exist on type 'Notification'`).
- `oxlint` — 0 warnings, 0 errors on all changed files.
- `oxfmt --check` (the repo's actual formatter, not Prettier) — all changed files correctly formatted.

### Full suite

Ran the full `yarn test` across all 9 workspaces — **all pass** (see Phase IV note about an environment issue I had to fix before this would run at all).

---

## Implementation Notes

### What I built

- `goal-template.ts`: added `values?: Record<string, unknown>` to the local `Notification` type; changed `` `Successfully applied templates to ${contexts.length} categories` `` to the static key `'Successfully applied templates to {{count}} categories'` plus `values: { count: contexts.length }`.
- `template-notes.ts`: added the same `values?` field (its "All templates passed! 🎉" and "There were errors interpreting some templates:" strings are static, so no message change needed — just shape consistency for the client union).
- `cleanup-template.ts`: added the same `values?` field, purely so the client-side notification type union compiles (its own strings were out of scope and left unchanged).
- `mutations.ts`: added a `translateBudgetTemplateMessage(t, notification)` helper — a `switch` over the four known message keys, each with a literal `t('...')` call so `i18next-parser` can extract them, with a `default` that passes any unrecognized message through untranslated. Wired it into `useBudgetActions`'s `onSuccess` in place of the raw passthrough. Added an explanatory comment stating _why_ the literal-`t()` seeding is necessary (loot-core can't call `t()`; the parser can't follow a variable key).
- `goal-template.test.ts`: one added assertion pinning the new `values: { count }` contract.
- `upcoming-release-notes/8413.md`: a `Bugfix` release note (the repo requires one per change).

### Challenges faced

- **The obvious fix is explicitly forbidden.** My first instinct — import `t` into `goal-template.ts` and wrap the strings — is exactly what PR #8027 removed on purpose, and the `i18next` dependency isn't even in `loot-core` anymore. I found this _before_ writing code by researching how the codebase handles server-originated translatable strings, which redirected the entire approach to the client boundary. Had I not checked, I'd have written a PR that a maintainer would (correctly) reject on sight.
- **Translating at runtime ≠ making translatable.** My first client-side attempt, a generic `t(notification.message, notification.values)`, _worked_ at runtime — but `i18next-parser` extracted none of the strings, so translators still couldn't see them and the "not on Weblate" complaint would remain unfixed. I only caught this by actually running `yarn generate:i18n` and grepping the output instead of assuming the runtime behavior was the whole story. The fix was to enumerate the known messages in a `switch` with literal `t()` calls the parser can statically see.
- **The type checker found a file the issue never mentions.** Adding `values` to two `Notification` types wasn't enough: the client union that `onSuccess` sees also includes the `cleanup-goal-template` action's notification, whose type is defined in a _third_ file (`cleanup-template.ts`). `yarn typecheck` failed with `Property 'values' does not exist on type 'Notification'` until I added the field there too. This is a good example of the fix's real surface area being wider than the issue's description — found only by compiling, not by reading.
- **Scoping the error block.** The `errors[]`→`pre` block mixes translatable UI copy with raw parser-error dumps. Rather than half-translate it or balloon the PR into a pipeline reshape, I made an explicit, documented decision to scope to the four named strings and leave the parser-error block in English. I confirmed that scope decision deliberately before implementing rather than discovering it mid-way.

### Code Changes

- **Files modified:**
  - `packages/desktop-client/src/budget/mutations.ts`
  - `packages/loot-core/src/server/budget/goal-template.ts`
  - `packages/loot-core/src/server/budget/goal-template.test.ts`
  - `packages/loot-core/src/server/budget/template-notes.ts`
  - `packages/loot-core/src/server/budget/cleanup-template.ts`
  - `upcoming-release-notes/8413.md` (new)
- **Commit:** [`691b583e1`](https://github.com/actualbudget/actual/commit/691b583e1203c6acde48c88b12f9fec48d3ce4fa) — `[AI] Make budget template notifications translatable` (branch `fix-issue-8413`)
- **Approach decisions:**
  - Translate at the **client** boundary (`mutations.ts`), never in `loot-core` — respecting PR #8027's "loot-core is i18n-free" architecture.
  - Server returns **stable keys + a `values` object**, not pre-formatted English sentences, so dynamic content (the category count) stays interpolatable and translatable.
  - A **literal-`t()` `switch`** rather than a generic `t(variable)` passthrough, so `i18next-parser` actually extracts the strings (fixing the "not on Weblate" half of the bug, not just the runtime half).
  - Scoped to the four issue-named strings; the mixed parser-error `pre` block was left English by explicit, documented decision.

### Phase IV Progress

**What I did before/around opening the PR:**

- Ran `yarn typecheck` (clean, 6 workspaces), `oxlint` (clean), and `oxfmt --check` (clean) on all changed files — noting the repo uses **oxlint/oxfmt**, not ESLint/Prettier, so I ran the project's actual tooling rather than generic formatters (bare `prettier --check` flagged false diffs from a different config; `oxfmt --check` reported clean).
- Ran `yarn generate:i18n` and grep-verified all four keys extracted into `locale/en.json`.
- Committed with the required `[AI]` prefix (Actual mandates an `[AI]` prefix on every AI-authored commit _and_ PR title — enforced for commits by a git-guard hook, and by hand for the PR title), pushed the branch to my fork, and opened PR #8417 against `actualbudget/actual`.
- Left the PR template **blank** on purpose: this repo's contributor rules explicitly say an AI agent must not fill in the PR template (Description/Testing/Checklist) — those sections are for the human who actually tested the change to complete.

**A real environment bug I had to fix before the suite would run:**

- After finishing the code, the repo's Stop-time check ran `yarn test` and it failed with `command not found: lage` / `command not found: vitest` — the task runner and test runner binaries weren't resolving at all.
- Rather than work around it, I root-caused it: `package.json` pinned `lage@^2.15.13` and `vitest@^4.1.8`, but the installed `node_modules` had **2.15.8** and **4.1.4** respectively — a stale install drifted behind the lockfile, so Yarn refused to expose the mismatched binaries. `yarn.lock` itself already had the correct resolved versions, confirming it was purely a `node_modules` sync problem (which is git-ignored and safe to rebuild).
- Ran `yarn install` to resync `node_modules` to the lockfile, then re-ran the full suite: `yarn test` now passes across all **9** workspaces, and `yarn typecheck` passes across all 6. No source change was needed for this — it was pure tooling drift, unrelated to my fix — but I verified it end-to-end rather than declaring the change done on top of a broken test runner.

---

## Pull Request

**PR Link:** [actualbudget/actual#8417](https://github.com/actualbudget/actual/pull/8417)

**What the PR does:** Makes the budget-template notifications ("Check templates," "Apply budget template," "Overwrite with budget template") translatable. The server now returns stable i18next message keys plus a `values` object for interpolation (e.g. category count); the client translates them via `t()` at the point notifications enter Redux, using explicit literal `t()` calls so `i18next-parser` can extract the strings into the translation catalog.

**Why it was needed:** The notification strings were hardcoded English literals built in `loot-core` (which is intentionally i18next-free per PR #8027) and shown to the client verbatim, so they never got translated and never appeared on Weblate for translators.

**Fixes:** #8413

**Acceptance:**

- [x] Test updated to pin the new server contract (`values: { count }`); all 194 `loot-core` budget tests pass.
- [x] All strings verified extracted into `locale/en.json` via `yarn generate:i18n`.
- [x] `yarn typecheck` / `oxlint` / `oxfmt` all clean; full `yarn test` (9 workspaces) passes.
- [x] Release note added (`upcoming-release-notes/8413.md`).
- [x] `loot-core` kept i18next-free, per PR #8027.

**Status:** Awaiting review. (The PR title currently shows a `[WIP]` prefix; the required `[AI]` prefix is present.)

---

## Learnings & Reflections

### Technical Skills Gained

- **How a "platform-agnostic core" constraint reshapes an i18n fix.** The single most important thing I learned here is that the _right_ place to fix a bug isn't always where the bug's symptom lives. The English strings are defined in `loot-core`, but a deliberate architecture decision (PR #8027) means they _cannot_ be translated there — so the fix belongs one layer out, at the client boundary. Reading the recent git/PR history for the affected files is what surfaced that.
- **The difference between "translated at runtime" and "extractable by tooling."** `t(variable)` translates but doesn't extract; `t('literal')` does both. For an i18n bug, the extraction half is the half that actually satisfies the "put it on Weblate" requirement, and it's the half that's easy to accidentally skip because the runtime behavior _looks_ correct.
- **Server APIs that need translation should return keys + params, not sentences.** Moving the category count out of the string and into a `values` object is what keeps a dynamic message both correct and translatable.

### Challenges Overcome

- Discovering the #8027 constraint _before_ writing the wrong fix, by researching the codebase's existing pattern for server-originated translatable strings instead of pattern-matching on "wrap it in `t()`."
- Catching, via `yarn generate:i18n`, that my first client-side attempt translated at runtime but extracted nothing — and reworking it into a literal-`t()` `switch`.
- Letting the type checker lead me to a third file (`cleanup-template.ts`) the issue never mentioned but the client type union required.

### What I'd Do Differently Next Time

I'd run the string-extraction command (`yarn generate:i18n`) as part of _defining_ the fix, not as a final verification step — because for an i18n issue, "does it extract?" is a design constraint on the very first line of code, not a check at the end. My generic-`t(variable)` first draft would never have been written if I'd treated extractability as a design requirement from the start rather than confirming it afterward.

### Connection to Contribution 1

Both contributions taught the same core lesson from opposite directions. In Contribution 1 (#8134), the fix I planned at the UI layer actually belonged in the server. Here, the fix I'd have planned at the server (`loot-core`) actually belonged at the client. In both cases, following the real data/dependency flow — and, this time, the recent architectural _decisions_ recorded in PR history — put the fix one layer away from where the symptom appeared.

---

## Resources Used

- [actualbudget/actual#8413](https://github.com/actualbudget/actual/issues/8413) — the issue.
- **PR #8027, "[AI] Move i18n usage outside of loot-core"** — the architectural decision that forbids the obvious fix and forces client-side translation; the single most important resource for this contribution.
- `packages/desktop-client/src/notifications/notificationsSlice.ts` — the existing precedent for translating a non-React-originated message via a module-level `t()`.
- `packages/desktop-client/i18next-parser.config.js` — confirmed the extractor scans `loot-core` source but can only extract literal `t('...')` calls, not `t(variable)`.
- `packages/desktop-client/src/budget/mutations.ts` — the client boundary where the fix ultimately lives.
- Actual's `AGENTS.md` / PR-and-commit rules — the `[AI]` prefix requirement, the "don't fill in the PR template" rule, and the run-yarn-from-root constraint.

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
