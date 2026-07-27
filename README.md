# AI Coding Skills — Setup & Usage

Four Copilot Agent Skills — a dev implementation pipeline, two codebase
summaries (backend + frontend), and one for revert decisions — plus a thin
`AGENTS.md` router:

| Skill | Purpose | Runs |
|---|---|---|
| `dev-workflow` | Requirement → Gap Check → Plan → Implement → Test → Summary, for any implementation task (Jira ticket or plain-text ask) | Once per task, invoked by the `AGENTS.md` router |
| `codebase-summary-kotlin-springboot` | Generate/update a compact architecture summary (`CODEBASE.md`) for a Kotlin/Spring Boot backend | Weekly / after big merges |
| `codebase-summary-nextjs` | Same idea, for a Next.js frontend — written in plain language for non-frontend readers | Weekly / after big merges |
| `requirement-version-resolver` | Resolves Jira/Confluence color-coded version markup into clean requirement text | Called from within `dev-workflow`, Step 1, only if source is Jira/Confluence |
| `revert-impact-assessment` | Decision-ready assessment for "can we revert this, what breaks, how long" | On demand, before a revert decision |

> Fullstack team with a separate backend and frontend repo? Install both
> codebase-summary skills — each repo gets its own `CODEBASE.md`.
>
> `AGENTS.md` stays deliberately thin (~120 words) — it's the only file
> loaded on every single message, so it only contains a router pointing to
> `dev-workflow` plus the Code Review section. Everything else lives in
> skills, loaded only when actually needed.

---

## 1. Install

All skills are folders containing a `SKILL.md` file. Copilot finds them by
**folder location**, not by file extension — a `.skill` file is just a zip
and must be extracted first.

### Where to put them

**Personal** (available in every repo you open, not shared with the team):
```
~/.copilot/skills/dev-workflow/SKILL.md
~/.copilot/skills/requirement-version-resolver/SKILL.md
~/.copilot/skills/codebase-summary-kotlin-springboot/SKILL.md
~/.copilot/skills/codebase-summary-nextjs/SKILL.md
~/.copilot/skills/revert-impact-assessment/SKILL.md
```

**Project** (shared with the team, version-controlled):
```
your-repo/.github/skills/dev-workflow/SKILL.md
your-repo/.github/skills/requirement-version-resolver/SKILL.md
your-repo/.github/skills/codebase-summary-kotlin-springboot/SKILL.md
your-repo/.github/skills/codebase-summary-nextjs/SKILL.md
your-repo/.github/skills/revert-impact-assessment/SKILL.md
your-repo/AGENTS.md
```

> `dev-workflow` should be **project** install (or at least consistent
> across the team) since it's the core pipeline everyone should follow the
> same way. `codebase-summary-*` skills are a good fit for **personal**
> install (install once, works in any repo of that stack).
> `revert-impact-assessment` is a good fit for **project** install too.

### If you have the `.zip`/`.skill` file
```bash
unzip dev-workflow.skill -d ~/.copilot/skills/
unzip requirement-version-resolver.skill -d ~/.copilot/skills/
unzip codebase-summary-kotlin-springboot.skill -d ~/.copilot/skills/
unzip codebase-summary-nextjs.skill -d ~/.copilot/skills/
unzip revert-impact-assessment.skill -d ~/.copilot/skills/
```

### If you have the plain `SKILL.md`
Just create the folder and drop the file in — folder name must match the
`name:` field in the file's frontmatter:
```
~/.copilot/skills/codebase-summary-kotlin-springboot/SKILL.md
```

### Verify
In Copilot Chat (VS Code), type:
```
/skills
```
All five skills should appear in the Configure Skills menu.

---

## 2. Skill: `dev-workflow`

### What it does
The core implementation pipeline: Requirement → Gap Check → AC extraction →
Plan → Implement → Test → Summary. Triggers for any task that implements,
fixes, or changes code based on a requirement — whether that requirement is
a Jira ticket or something typed directly in chat.

Key behaviors:
- Extracts Acceptance Criteria as a checklist and **persists it to
  `.task/{slug}-ac.md`** as checkboxes so it survives context compaction on
  long tasks — then **writes the pass/fail status back into that same file**
  after testing (auto-verified vs needs-manual-verification), so the file
  is actually useful when the user opens it later, not just a stale copy of
  the original list
- Only stops to ask questions for genuine blockers (spec/code conflicts) —
  infers and states assumptions for everything else
- Skips Plan confirmation entirely for small, self-contained changes
- Test step retries failures up to 2 times, then stops and reports rather
  than looping forever
- Copies the final AC pass/fail table into the PR description, then
  **keeps** `.task/{slug}-ac.md` around (doesn't auto-delete) — items marked
  "needs manual verification" need the file to still exist so the user can
  come back and check them off after the task is done

### Usage
Don't call this directly by name most of the time — the `AGENTS.md` router
invokes it automatically whenever a task looks like implementation work.
You can still invoke it explicitly:
```
run dev-workflow for: add retry logic to the payment webhook
```
Or just describe the task normally in chat — the router in `AGENTS.md`
checks every task-like message and invokes this skill itself.

### Setup requirement
Add `.task/` to `.gitignore` — these files are local scratch (survive across
the task and stick around afterward for manual verification), never meant
to be committed. Clean them up yourself once you've finished verifying;
`dev-workflow` won't delete them for you.

---

## 3. Skill: `requirement-version-resolver`

### What it does
Fetches a requirement from Jira or Confluence and resolves color/strikethrough
version markup (green = added in version X, red strikethrough = removed in
version Y, or whatever convention your team uses) down to plain, clean text
for the target version — instead of an agent trying to interpret color-coded
HTML directly and guessing wrong.

Called automatically from within `dev-workflow`, Step 1, whenever the
requirement source is Jira/Confluence — you shouldn't need to invoke it by
name separately.

### Important notes
- Depends on your Jira/Confluence MCP connector returning raw content (ADF
  or storage format), not a pre-rendered summary — check what MCP tools are
  actually connected if this skill reports it can't verify version markup
- If a page has no legend explaining what each color means, it will stop
  and ask once rather than guessing the convention
- Caches resolved output by source ID + version, so re-running on an
  unchanged page/issue doesn't re-parse it

---

## 4. Skill: `codebase-summary-kotlin-springboot`

### What it does
Scans a Kotlin/Spring Boot repo once and produces `docs/codebase/CODEBASE.md`
— stack, layering, API surface, domain/service, data layer, conventions, and
watch-out areas. Subsequent runs auto-detect whether to do a full scan or a
cheap diff-based update, and link the result into `AGENTS.md`/`CLAUDE.md`.

### Recommended model
Use **Claude Sonnet 4.6 or Claude Sonnet 5** — this task needs real
architecture comprehension, not just pattern matching. Since it doesn't run
every task (only weekly/after merges), the extra cost per run is worth it for
accuracy. Avoid mini/fast-tier models here (GPT-5.4-mini, Grok Code Fast,
Claude Haiku) — higher risk of missing subtle conventions.

### Usage

**First time (no CODEBASE.md yet):**
```
generate codebase summary
```
This also creates/links `AGENTS.md` (or `CLAUDE.md` if that's what you use)
automatically — no manual setup needed after this.

**After implementing a change:**
```
update codebase summary
```
The skill checks the existing file's commit sha, diffs only what changed
since then, and patches the relevant sections. If changes are too large
(>30% of the repo, or the layering structure itself changed), it will
recommend a full re-generate instead of patching.

### Output location
```
docs/codebase/CODEBASE.md              (or split per-module for large repos)
docs/codebase/summary-<module>.md
```

---

## 5. Skill: `codebase-summary-nextjs`

### What it does
Same idea as the Kotlin skill, adapted for a Next.js frontend — stack,
routing, rendering model (Server vs Client Components), component/data
layer, conventions, and watch-out areas. Written specifically to be
readable without a frontend background: unusual terms get a short plain-
language explanation instead of being assumed knowledge. Auto-detects App
Router vs Pages Router vs a mixed migration-in-progress state, and flags
security-relevant mistakes like secrets accidentally exposed via
`NEXT_PUBLIC_` environment variables.

### Recommended model
Same guidance as the Kotlin skill — **Claude Sonnet 4.6 or Claude Sonnet 5**.
Router/rendering-model detection needs real comprehension, not just pattern
matching, and this runs infrequently enough that the extra cost is worth it.

### Usage

**First time (no CODEBASE.md yet):**
```
generate codebase summary
```

**After implementing a change:**
```
update codebase summary
```
Same diff-based behavior as the Kotlin skill: checks the last commit sha,
patches only affected sections, and falls back to a full re-generate if the
routing model itself changed significantly (e.g. Pages→App migration
progressed a lot) or changes exceed ~30% of the repo.

### Output location
```
docs/codebase/CODEBASE.md              (or split per-module for large repos)
docs/codebase/summary-<module>.md
```

---

## 6. Skill: `revert-impact-assessment`

### What it does
Given a commit, PR, or feature, produces a one-page assessment: can it be
reverted cleanly, what breaks (code/API/data), how long it'll take, the
least-risky way to do it, and whether later commits depend on it. Built for
bringing a clear answer into a meeting — not a deep code walkthrough.

### Scope — important
Only works on commits that have actually **shipped** (merged into
`master`/`main` or the active release branch). If a commit only lives on an
unmerged feature branch, the skill will say so and suggest just closing the
PR instead — there's no real "impact" to assess yet.

### Usage

**Single commit:**
```
revert impact for a1b2c3d
```

**Multiple commits via a PR:**
```
revert impact for PR #482
```

**No sha, just a rough description:**
```
revert impact for the payment retry feature, shipped around mid-July
```
The skill will search git history for candidates and confirm the commit list
with you before running the analysis.

### Safety
Runs `git revert --no-commit` as a dry-run only, then immediately aborts —
never touches commit history, never pushes, never affects the real codebase.
It does require a clean working tree first (commit or stash pending changes),
and will ask before proceeding if it isn't.

### Output
```markdown
# Revert Assessment: {feature/commit}
> Branch analyzed: {master/main or release/x.y}

## Summary — one line, decision-ready
## Feasibility — Clean / Conflict / Not revertable
## Impact — Code / API / Data
## Timeline — Revert time / Test-QA time (separate)
## Recommended approach + fallback
## Later-commit entanglement — Independent / Entangled
```

---

## 7. Model tiering

This lives here, not in `AGENTS.md` or the skill files — the AI itself
doesn't act on this info (a human picks the model, or automation controls
it), so keeping it in an always-loaded file would just spend tokens every
request for no benefit.

| Runs | Model |
|---|---|
| `dev-workflow` Step 1 (restate, AC extraction, blocker classification) | Light (Haiku 4.5 / GPT-5.4-mini) |
| `dev-workflow` Step 2+3 Plan | Light-to-mid; escalate to Sonnet if the change is cross-module |
| `dev-workflow` Step 4 Implement | Strongest available — this step's output quality matters most |
| `dev-workflow` Step 5 Test / AC cross-check | Light |
| `codebase-summary-*`, `requirement-version-resolver`, `revert-impact-assessment` | Claude Sonnet 4.6 / Sonnet 5 — infrequent but needs real comprehension, not pattern matching |

---

## 8. How AGENTS.md and the skills fit together

`AGENTS.md` is deliberately kept to ~120 words because it's the one file
loaded on **every single message**, task-related or not. It contains only:

```markdown
## Router
For any task that involves writing or modifying code based on a
requirement — a Jira ticket, a PR, or a requirement typed directly in
chat — invoke the `dev-workflow` skill explicitly at the start of the task
and follow it end-to-end. Don't rely only on automatic skill-matching;
check for this case directly. Skip this for pure questions, read-only
exploration, or discussion that isn't asking for a code change.

## Code Review
Copilot code review reads this file on every PR...
```

Everything else — the actual 6-step pipeline, AC checklist handling, gap
checking — lives in `dev-workflow`, which only loads when a task is
actually being implemented, not on every message in the conversation. This
is the same principle as `CODEBASE.md` vs `codebase-summary-*`: keep the
always-loaded layer thin, push detail into on-demand skills.

```
message unrelated to coding  → AGENTS.md router only (~150 tokens)
message = implement a ticket → AGENTS.md router → invokes dev-workflow
                                  → full pipeline runs, loaded once for
                                    this task, not reloaded every message
                                    within it
```

---

## 9. Troubleshooting

| Problem | Fix |
|---|---|
| `.skill` file won't open | It's a zip — `unzip file.skill -d <destination>` |
| Skill not showing in `/skills` menu | Check folder name matches `name:` in frontmatter exactly; confirm it's in `~/.copilot/skills/` or `.github/skills/` |
| Model not selectable in chat | Business/Enterprise orgs need an admin to enable it under Copilot policy settings |
| Skill runs but seems to ignore instructions | Confirm you're on the correct branch — `revert-impact-assessment` in particular depends on being anchored to a shipped branch (master/release), not a feature branch |
| `dev-workflow` never triggers on a plain-text ask | Check the `AGENTS.md` router is actually present in the repo — without it, skill auto-discovery alone may miss casually-phrased requests |
| `.task/*-ac.md` files show up in `git status` | Add `.task/` to `.gitignore` |