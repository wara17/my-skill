---
name: "dev-workflow"
description: "Use this skill when starting to implement, fix, or change code based on a requirement — whether that requirement comes from a Jira ticket, a PR description, or plain text typed directly by the user in chat (e.g. \"add a retry to the payment webhook\", \"fix the null pointer in OrderService\"). This is the standard implementation pipeline covering requirement understanding, gap check, planning, implementation, testing, and summary. Do NOT trigger for pure questions, read-only exploration, or discussions that aren't asking for a code change."
---

# Dev Workflow Skill

## Goal
One pipeline for implementing any requirement, regardless of where it came
from. Runs once per task/ticket — not reloaded on every message within the
same task, and not loaded at all for messages unrelated to implementation
work (questions, exploration, casual conversation).

**Steps below are the default flow, not a locked sequence.** If the user
explicitly asks to skip ahead or run a single step in isolation (e.g. "just
implement this, skip planning" / "just run the AC check"), follow that
instruction directly instead of forcing the full pipeline.

---

## Step 1 — Requirement, Gap Check, AC extraction
**Sub-steps 1-6 below are never skipped — not for bug fixes, not for small
changes, not for anything.** The only thing that's ever conditional is
whether Step 7 stops to wait for plan confirmation. "Fast path" means skip
the *stop*, not skip the *work*. If Step 1 finishes without a `.task/*.md`
file existing on disk, that's a failure to follow this skill correctly,
regardless of how simple the task seemed.

1. **Identify the source**:
   - Jira ticket / Confluence page → run `requirement-version-resolver`
     first to resolve color/strikethrough version markup to clean text
   - Plain text typed by the user in chat → use it directly, no resolver
     needed
2. Restate the requirement in 1-2 sentences.
3. **Extract Acceptance Criteria as a checklist**, separate from prose.
   **Bug fixes have AC too** — even a one-line bug fix has implicit AC, e.g.
   "the reported error no longer occurs," "existing behavior for other
   inputs is unchanged," "no new test regressions." Never skip AC
   extraction just because the task is a bug fix or looks small.
   AC not explicitly stated → infer and mark as an assumption (same
   blocker/no-blocker rule as below applies to inferred AC). **Every
   inferred assumption goes into a visible "Assumptions" list — never
   inferred silently and folded straight into implementation.** The user
   needs to see what was guessed before code gets written on top of it.
4. **Gap Check**: cross-reference against `docs/codebase/CODEBASE.md` (or
   relevant sections of it) and any files the user already named. Do not
   scan the whole repo if the file(s) needed are already known and the
   change is self-contained. If `docs/codebase/CODEBASE.md` doesn't exist at
   all yet, run the matching `codebase-summary-*` skill first — don't
   attempt the Gap Check against a file that isn't there.
5. **Persist immediately, always**: write the AC checklist to
   `.task/{branch-or-ticket-slug}-ac.md` as markdown checkboxes (`- [ ] ...`),
   unchecked. This happens on every task, including the fast path in step 7
   below — this is the source of truth for Steps 2-6, re-read it from the
   file at each checkpoint, don't rely on conversation memory (context can
   get compacted on long tasks, and Copilot's own auto-compaction has been
   known to drop skill instructions from context entirely on longer tasks —
   the file is the only thing guaranteed to survive that).
6. **Blocker rule** — stop and ask (once, all questions combined into one
   message) only if guessing wrong would cause real damage:
   - Blocker: spec conflicts with a DB/type constraint (e.g. optional field,
     NOT NULL column, no default)
   - Blocker: requirement contradicts something CODEBASE.md's "Watch out"
     section warns about
   - Blocker: requirement references a component/flow that doesn't exist
   - Not a blocker: unstated error message wording, minor edge cases that
     don't affect data integrity — infer from existing patterns, state the
     assumption, keep going.
7. **Only now, decide whether to stop for plan confirmation** (sub-steps
   1-6 above already happened regardless of this decision — the AC file
   already exists on disk either way):
   - No blockers, AC clear, change is self-contained (single file/module,
     no schema change, no cross-service impact) → continue directly into
     Explore + Plan + Implement in the same response, no confirmation stop.
     **Still open the response with the Assumptions list from step 3** (even
     if short, even if just "none") before showing the plan/code — the user
     should never have to dig for what was guessed.
   - Otherwise → proceed to Step 2+3 and stop for Plan confirmation.

## Step 2+3 — Explore + Plan (skip if Step 1 already routed to fast path)
- Read `docs/codebase/CODEBASE.md` (only the relevant sections) plus any
  files it points to that are actually needed — don't explore beyond that
  without a reason.
- **Open with the Gaps/Assumptions list carried over from Step 1** — what's
  unclear, what's being guessed and why, what a blocker forced you to ask
  about. This is what the user is actually confirming, not just the plan
  bullets that follow from it.
- Write the plan as short bullets, each one referencing what it's based on
  (e.g. "per CODEBASE.md's error-handling convention" / "per
  OrderService.kt"). A plan bullet with no traceable source is invalid.
- Map each plan item to the AC item(s) it satisfies.
- **Include, as an explicit plan item, which tests will be written or
  updated for each new/changed AC item** — this isn't optional detail, it's
  what makes Step 5b's auto-verification possible later. A plan with code
  changes but no corresponding test changes should be treated as incomplete.
- Stop here and wait for plan confirmation.

## Step 4 — Implement
- Follow the confirmed plan and existing conventions.
- **Write or update unit tests covering each new or changed AC item as part
  of this step, not as an afterthought.** Step 5 only runs the existing test
  suite — it does not write new tests. If no test exists for a given AC item
  by the time Step 5 runs, that item cannot be auto-verified and will fall
  through to "needs manual verification" even if the code is correct. Tests
  written here are what make Step 5b's pass/fail table meaningful instead of
  mostly empty.
- If this task exceeds ~10 tool calls or touches more than ~5 files,
  re-read `.task/{slug}-ac.md` from disk mid-task before continuing —
  don't trust what's still in context.

## Step 5 — Test
5a. **Lint/format first, if the project has one configured.** Check
    CODEBASE.md's Conventions section for a lint/format command (ktlint,
    detekt, spotless, etc.) and run it, fixing violations directly. This is
    cheap to do now and avoids a second review round purely over style.
5b. Run the test suite using the command noted in CODEBASE.md's Testing
    section (don't assume `npm test`/`gradle test` — check the file; if the
    Testing section is missing because CODEBASE.md predates this being
    tracked, ask the user for the command once rather than guessing). If
    it fails, fix and retry up to **2 times**. Still failing after that →
    stop, report the error, let the user decide. Don't retry indefinitely.
5c. Re-read `.task/{slug}-ac.md` from disk and check off each AC item
    individually — pass/fail per item, not one overall "looks good."
    **Update the file itself**, not just the response: mark each item
    `- [x] {item} — auto-verified` if a test covers it and passes,
    or leave `- [ ] {item} — needs manual verification` if nothing covers
    it. This is what makes the persisted file useful afterward — the user
    should be able to open `.task/{slug}-ac.md` later and see exactly
    what's already confirmed vs what they still need to check themselves.
    A high proportion of "needs manual verification" items is a signal that
    Step 4's test-writing was skipped or incomplete — worth flagging to the
    user rather than passing over silently.

## Step 6 — Summary
- List files changed, 1-3 lines.
- Copy the AC pass/fail table from Step 5 into the PR description. This
  skill prepares that description but does not push commits or open the PR
  itself — say so explicitly rather than implying it's been done, so the
  user knows that part is still theirs to do (or ask them how they'd like
  that handled, if it's unclear).
- **Do NOT delete `.task/{slug}-ac.md`.** Keep it — it now holds the
  updated checklist from Step 5c (auto-verified vs needs-manual-verification),
  not the raw Step 1 version. Cleanup is manual: the user deletes it
  themselves once they've finished verifying, not the agent.

## Related skills used within this pipeline
| Skill | Called from |
|---|---|
| `requirement-version-resolver` | Step 1, only if source is Jira/Confluence |
| `codebase-summary-kotlin-springboot` / `codebase-summary-nextjs` | Step 1 (if CODEBASE.md missing) and Step 2+3, whichever matches the repo's stack |

Model selection per step lives in the team README, not here — it's a
human/admin decision, not something this file needs to spend tokens on.

