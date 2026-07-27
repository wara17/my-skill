---
name: revert-impact-assessment
description: Use this skill when someone asks whether a feature, commit, or PR
  can be reverted/rolled back, or wants an impact/timeline assessment before
  removing something — e.g. "can we revert this", "impact of removing feature X",
  "rollback assessment", "เอาออกได้ไหม", "ถอด feature นี้กระทบอะไรบ้าง". This is
  a fast, decision-ready assessment for meetings, NOT a deep business-logic
  reverse-engineering (that's a separate skill).
---

# Revert Impact Assessment Skill

## Goal
Produce a one-page, decision-ready answer to: can this be reverted, what
breaks if we do, how long will it take, and what's the least-risky way to do
it. Output is for a meeting — leadership should be able to read the summary
line and act on it. Do NOT do a full business-logic reverse-engineer here;
that's slower and belongs to a different skill.

---

## Step 0 — Identify the target
Accept any of these input shapes — resolve to a list of commit shas
(chronological order, oldest→newest) before moving on:
- **Single commit sha** → the list is just that one sha
- **PR number** → resolve via `gh pr view <number> --json commits -q '.commits[].oid'`
  (or `git log --oneline <base>..<head>` if `gh` isn't available)
- **Feature name / rough date, no sha given** → `git log --grep="<keyword>"` or
  `git log --since --until` to find candidates, then confirm the list with the
  user before proceeding — don't guess which commits belong to the feature
- If genuinely ambiguous, ask for a commit sha, PR number, or date range
  rather than guessing.

---

## Step 1 — Feasibility check
- Dry-run only, never leave the repo in a dirty state:
  - **Single commit**: `git revert --no-commit <sha>`
  - **Contiguous range**: `git revert --no-commit <oldest>^..<newest>` — git
    applies them in the correct order automatically
  - **Non-contiguous list** (e.g. PR with commits from different branches
    merged in): `git revert --no-commit <newest> <next> ... <oldest>` — list
    shas newest-first, git reverts them in the order given
  - Immediately `git revert --abort` (or `git reset --merge`) once you've
    read the result — never leave it uncommitted.
- Classify into exactly one of:
  - **Clean** — the whole set applies with no conflicts
  - **Conflict** — note which specific commit(s) in the set conflict, not just
    "some conflict" — e.g. "3 of 5 commits clean, 2 conflict on OrderService.kt"
  - **Not revertable** — structural/schema changes since then make a straight
    revert meaningless (see Step 2, Data-level)

## Step 2 — Impact, in three layers
Use the **union of all files** touched across every commit in the set (not
just one commit) as the basis for this step.
- **Code-level**: `git grep` / IDE "find usages" for the functions/classes
  touched — list every caller, not just the file that was edited.
- **API-level**: is any changed endpoint/contract called by something outside
  this repo (external consumer, mobile app, another service)? If yes, revert
  = potential breaking change, flag it explicitly.
- **Data-level**: did this change add/alter a DB column, migration, or produce
  data that now depends on the new behavior? If real data already exists under
  the new behavior, a code-only revert can desync from the schema/data —
  this is usually the single biggest reason something becomes "Not revertable."

## Step 3 — Timeline (always split into two numbers)
- **Revert execution time**: minutes-to-hours if Clean; longer if Conflict
- **Test/QA time**: usually the larger number — regression scope depends on
  how many callers/consumers Step 2 found. Call this out separately since
  it's the number people usually forget to ask about.

## Step 4 — Solutions, ranked by blast radius (least → most impact)
Always produce this table, even if only 1-2 rows apply:

| Option | Impact | When it applies |
|---|---|---|
| Flip a feature flag off | Near-zero, instantly reversible | Only if the change shipped behind a flag |
| Straight `git revert` | Low, if Clean | No later commits touch the same files |
| Partial/manual revert | Medium | Conflict case — some later changes must be preserved |
| New migration/compensating change instead of revert | High | Data-level already diverged — Step 2 flagged "Not revertable" |

## Step 5 — Entanglement with later commits (the meeting question)
- Anchor on the **newest commit in the set** (everything after the whole
  feature landed counts as "later"): `git log <newest>..HEAD --oneline -- <union of affected files>`
- Classify:
  - **Independent** — nothing later touches these files → straightforward revert
  - **Entangled** — list every later commit that touches the same files, so
    the room can see exactly what else would be dragged in or broken. This
    directly answers "if we remove this, does it break feature Y that shipped
    after it?"

---

## Output template

```markdown
# Revert Assessment: {feature/commit}

## Summary (one line, decision-ready)
[Recommend revert / Do not recommend / Revert possible with conditions]

## Feasibility: Clean / Conflict / Not revertable
{one or two sentences why}

## Impact
- Code: {callers/modules affected}
- API: {breaking change or not}
- Data: {schema/data dependency status}

## Timeline
- Revert: {estimate}
- Test/QA: {estimate}

## Recommended approach (+ fallback)
1. {least-impact viable option from Step 4}
2. {fallback if #1 turns out not to apply}

## Later-commit entanglement
[Independent / Entangled — list commits if entangled]
```

## Guardrails
- Before Step 1, confirm the working tree is clean (`git status`). If there
  are uncommitted changes, stop and ask the user to commit or stash them
  first — a dry-run revert on a dirty tree can interact with unsaved work.
- Never leave a dry-run revert uncommitted-but-unresolved in the working tree
  — always abort/reset after inspecting it.
- Don't estimate Test/QA time as a guess — base it on the number of callers
  and consumers found in Step 2; if that number is large, say "needs QA input"
  rather than inventing a figure.
- If Step 2 or Step 5 turns up anything ambiguous (unclear consumer, unclear
  data dependency), state the uncertainty in the Summary rather than picking
  an optimistic answer — this output gets used to make a real decision.
