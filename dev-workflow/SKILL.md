---
name: dev-workflow
description: >-
  Use this skill when starting to implement, fix, or change code
  based on a requirement — whether that requirement comes from a Jira ticket,
  a PR description, or plain text typed directly by the user in chat (e.g.
  "add a retry to the payment webhook", "fix the null pointer in
  OrderService"). This is the standard implementation pipeline covering
  requirement understanding, gap check, planning, implementation, testing,
  and summary. Do NOT trigger for pure questions, read-only exploration, or
  discussions that aren't asking for a code change.
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
   risk-tagging rule in step 6 below applies to inferred AC too). **Every
   inferred assumption goes into a visible "Assumptions" list — never
   inferred silently and folded straight into implementation.** The user
   needs to see what was guessed before code gets written on top of it.
4. **Gap Check**: cross-reference against `docs/codebase/CODEBASE.md` (or
   relevant sections of it) and any files the user already named. Do not
   scan the whole repo if the file(s) needed are already known and the
   change is self-contained.
5. **Persist immediately, always**: write the AC checklist to
   `.task/{branch-or-ticket-slug}-ac.md` as markdown checkboxes (`- [ ] ...`),
   unchecked. This happens on every task, including the fast path in step 7
   below — this is the source of truth for Steps 2-6, re-read it from the
   file at each checkpoint, don't rely on conversation memory (context can
   get compacted on long tasks, and Copilot's own auto-compaction has been
   known to drop skill instructions from context entirely on longer tasks —
   the file is the only thing guaranteed to survive that).
6. **Unknown surfacing** — compile every unclear/unstated item found during
   Gap Check into a single list, tag each by risk:
   - 🔴 **High-risk** — a guess here isn't cheaply fixed later: conflicts
     with a DB/type constraint, contradicts CODEBASE.md's "Watch out",
     references a component/flow that doesn't exist, or omits a structural
     identifier/contract value (error code, status code, event name, enum
     value, API field name) that other systems might depend on
   - 🟡 **Low-risk** — free text, wording, labels, cosmetic edge cases that
     don't touch any identifier or data integrity
   - **🟡 items never block.** Infer automatically, state the assumption in
     the visible Assumptions list, keep going — no round-trip spent on these.
   - **If any 🔴 item exists, stop and present the full list with a choice
     per item** (or "same choice for all" if the user says so):
     1. **Implement what's known** — build the clear parts normally, leave
        an explicit `TODO`/stub for each 🔴 unknown instead of guessing its
        value
     2. **Infer and proceed** — guess reasonably, state the assumption
        clearly, keep going (same as how 🟡 items are always handled)
     3. **I'll provide it** — user adds the missing detail; re-check *only*
        that item against Gap Check, don't re-run Step 1 from scratch
   - Combine all 🔴 questions into one message — never ask one at a time.
7. **Only now, decide whether to stop for plan confirmation** (sub-steps
   1-6 above already happened regardless of this decision — the AC file
   already exists on disk either way, and any 🔴 unknowns were already
   resolved via the 3-choice menu before reaching this point):
   - No unresolved 🔴 items, AC clear, change is self-contained (single
     file/module, no schema change, no cross-service impact) → continue
     directly into Explore + Plan + Implement in the same response, no
     confirmation stop. **Still open the response with the Assumptions
     list from step 3** (even if short, even if just "none") before
     showing the plan/code — the user should never have to dig for what
     was guessed.
   - Otherwise → proceed to Step 2+3 and stop for Plan confirmation.

## Step 2+3 — Explore + Plan (skip if Step 1 already routed to fast path)
- Read `docs/codebase/CODEBASE.md` (only the relevant sections) plus any
  files it points to that are actually needed — don't explore beyond that
  without a reason.
- **Open with the Gaps/Assumptions list carried over from Step 1** — what's
  unclear, what's being guessed and why, which 🔴 items were resolved via
  the 3-choice menu and what was chosen. This is what the user is actually
  confirming, not just the plan bullets that follow from it.
- Write the plan as short bullets, each one referencing what it's based on
  (e.g. "per CODEBASE.md's error-handling convention" / "per
  OrderService.kt"). A plan bullet with no traceable source is invalid.
- Map each plan item to the AC item(s) it satisfies.
- Stop here and wait for plan confirmation.

## Step 4 — Implement
- Follow the confirmed plan and existing conventions.
- If this task exceeds ~10 tool calls or touches more than ~5 files,
  re-read `.task/{slug}-ac.md` from disk mid-task before continuing —
  don't trust what's still in context.

## Step 5 — Test
5a. Run the test suite using the command noted in CODEBASE.md's Testing
    section (don't assume `npm test`/`gradle test` — check the file). If
    it fails, fix and retry up to **2 times**. Still failing after that →
    stop, report the error, let the user decide. Don't retry indefinitely.
5b. Re-read `.task/{slug}-ac.md` from disk and check off each AC item
    individually — pass/fail per item, not one overall "looks good."
    **Update the file itself**, not just the response: mark each item
    `- [x] {item} — auto-verified` if the test suite covers it and passes,
    or leave `- [ ] {item} — needs manual verification` if nothing covers
    it. This is what makes the persisted file useful afterward — the user
    should be able to open `.task/{slug}-ac.md` later and see exactly
    what's already confirmed vs what they still need to check themselves.

## Step 6 — Summary
- List files changed, 1-3 lines.
- Copy the AC pass/fail table from Step 5 into the PR description.
- **Do NOT delete `.task/{slug}-ac.md`.** Keep it — it now holds the
  updated checklist from Step 5b (auto-verified vs needs-manual-verification),
  not the raw Step 1 version. Cleanup is manual: the user deletes it
  themselves once they've finished verifying, not the agent.

## Related skills used within this pipeline
| Skill | Called from |
|---|---|
| `requirement-version-resolver` | Step 1, only if source is Jira/Confluence |
| `codebase-summary-*` | Step 2+3, whichever matches this repo's stack (Kotlin/Spring Boot, Next.js, Go, etc.) |

Model selection per step lives in the team README, not here — it's a
human/admin decision, not something this file needs to spend tokens on.
