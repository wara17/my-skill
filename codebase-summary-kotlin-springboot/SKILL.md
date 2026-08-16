---
name: "codebase-summary-kotlin-springboot"
description: "Use this skill when the user asks to generate, update, or refresh a codebase summary for a Kotlin + Spring Boot project — e.g. \"summarize codebase\", \"update CODEBASE.md\", \"generate codebase summary\", \"summarize this repo\". Also trigger automatically before any task that needs project-wide context (new feature, refactor, bug investigation) if CODEBASE.md doesn't exist yet or is older than the latest relevant commits."
---

# Codebase Summary Skill — Kotlin / Spring Boot

## Goal
Scan the repo once (pay the cost once) and produce a `CODEBASE.md` file compact
enough that other agent tasks can load it instead of re-indexing the whole repo.
This is NOT a detailed reference doc.

**Do NOT**: copy full source files, explain every function line-by-line, or
exceed ~150 lines. If the repo is large, split into per-module files instead
of stuffing everything into one (see "Large repos" below).

---

## Mode check (do this first, before anything else)
- Check whether `docs/codebase/CODEBASE.md` (or `CODEBASE.md` at repo root)
  already exists.
- **Doesn't exist → Generate mode**: run steps 1-7 below in full, produce the
  file from scratch using the Output template.
- **Exists → Update mode**: skip steps 1-7 entirely, jump straight to the
  "Updating" section near the end of this file. Never redo a full scan just
  because the user said "update" — that defeats the point of diffing.

---

## Scan steps (Generate mode only — skip if Update mode)

### 1. Build & config
- Read `build.gradle.kts` (or `pom.xml`) → Kotlin/Spring Boot version, key
  dependencies (web/webflux, jpa, security, kafka, redis, etc.)
- Read `application.yml`/`application.properties` and profile variants
  (`-dev`, `-prod`, `-test`) → note only keys that affect behavior (never secrets)
- Identify build tool + multi-module structure if any (`settings.gradle.kts`)
- Get the commit the summary is generated from: run `git rev-parse --short HEAD`
  and use that literal value for `{short_sha}` in the Output template header —
  don't guess or leave it as a placeholder.

### 2. Entry point & layering
- Find the `@SpringBootApplication` class → root package
- Identify the actual layering convention used (not theory), e.g.
  `controller / service / repository / domain / dto` or hexagonal
  (`adapter / domain / application`) — infer from real folder structure

### 3. API layer
- Scan `@RestController` / `@Controller` → list main endpoints (method + path),
  grouped by resource if there are dozens rather than listing every one
- Find `@ControllerAdvice` / `@ExceptionHandler` → the actual error-handling
  pattern in use
- Check whether the project uses MVC (blocking) or WebFlux (`Mono`/`Flux`,
  coroutines) — this matters a lot since it shapes everything the agent writes

### 4. Service & domain layer
- Scan `@Service` classes → one-line responsibility per class (skip internal logic)
- Find domain models: `data class`, `sealed class`, `enum class` that represent
  core business concepts
- Note Kotlin-specific patterns actually used by the team: null-safety style,
  extension functions, coroutines (`suspend fun`) vs blocking, constructor
  injection (near-universal in Kotlin projects)

### 5. Data layer
- Scan `@Entity` / `@Table` → main entities and their relationships, briefly
  (no need for a full ERD)
- Find repositories (`JpaRepository`, `CoroutineCrudRepository`, custom `@Query`)
- Check migration tool (Flyway `db/migration` or Liquibase) → note latest
  migration file location

### 6. Cross-cutting concerns
- Security: `SecurityFilterChain` config, auth mechanism (JWT/OAuth2/session)
- External integrations: Kafka topics, Redis usage, external REST clients
  (`WebClient`/`Feign`)
- Testing setup: JUnit5 + MockK/Mockito, Testcontainers usage, test naming
  convention. **Also capture the literal, runnable command used to execute
  tests** (e.g. `./gradlew test`, or a module-scoped variant for multi-module
  repos) — check the Gradle/Maven config or CI pipeline file for the exact
  command actually used, don't infer "JUnit5" and assume a generic command.
  This is what the `dev-workflow` skill's Test step reads directly, so it
  needs to be literal and copy-pasteable, not a description.
- This step's findings map to the **Testing** and **Security & Integrations**
  sections of the Output template below — don't gather this and then drop it,
  every item scanned here needs a home in the output file.

### 7. Conventions (the most important part for the agent to capture)
- Real naming conventions used in this project (not the generic Kotlin style guide)
- Standard error/response handling pattern the team actually follows
- Lint/format tooling in use, if any (ktlint, detekt, spotless) and the command
  to run it — later implementation work should run this before considering a
  task done, so it needs to be discoverable from here
- Areas that are "hands off" — legacy code or not-yet-migrated parts

---

## Output template

```markdown
# CODEBASE.md
> Last updated: {date} — generated from commit {short_sha}

## Stack
- Kotlin {version} / Spring Boot {version} / {MVC or WebFlux}
- Build: {gradle/maven}, multi-module: {yes/no}
- DB: {postgres/mysql/...}, migration: {flyway/liquibase}

## Structure
{main folders + one-line purpose each}

## API surface
{endpoints grouped by resource, not every single one if there are many}

## Domain & Service
{main services + responsibility, main domain models}

## Data layer
{main entities + relationships, briefly}

## Testing
- Framework: {JUnit5 + MockK/Mockito, Testcontainers: yes/no}
- Run command: {literal command, e.g. `./gradlew test`}
- Naming convention: {e.g. `should_doX_when_Y`}

## Security & Integrations
- Auth: {JWT/OAuth2/session, where configured}
- External systems: {Kafka topics, Redis usage, WebClient/Feign clients}

## Conventions
{naming, error handling, DI style, null-safety pattern, lint/format command}

## ⚠️ Watch out
{legacy code, off-limits areas, complex business logic to be careful with}
```

---

## Large repos (multi-module or >~50 files per module)
Split into smaller files instead of one giant file:
```
docs/codebase/CODEBASE.md          ← overview, links to sub-summaries
docs/codebase/summary-auth.md
docs/codebase/summary-payment.md
docs/codebase/summary-notification.md
```
The agent loads only the file relevant to the current task, not everything.
Each sub-summary should still carry its own `## Testing` section if the
module has module-scoped test commands (common in multi-module Gradle repos)
— don't assume the root-level test command applies everywhere.

---

## Updating (never regenerate everything from scratch each time)
- Trigger an update: after a large PR merge, or weekly — not on every task
- Read the commit sha from the existing file's `Last updated` header, then run
  `git diff {last_sha}..HEAD --stat` — commit-based, not date-based, since
  `--since` is prone to timezone and commit-timing mismatches
- Map changed files to the relevant section(s) above (Controller changes → API
  surface, Entity changes → Data layer, build file changes → Testing/Stack,
  security config changes → Security & Integrations, etc.) and patch only
  those sections
- **Safety threshold**: if changed files exceed ~30% of the repo, or the
  layering/module structure itself changed, STOP patching and recommend
  running Generate mode instead — piecemeal patching at that scale risks an
  inconsistent file
- Bump `Last updated` date + commit sha (via `git rev-parse --short HEAD`) in
  the header every time it's edited

## Step 8 (Generate mode only — skip in Update mode): Link from AGENTS.md/CLAUDE.md
After producing CODEBASE.md for the first time, check the repo root for
`AGENTS.md` or `CLAUDE.md`:
- **Neither exists** → create `AGENTS.md` (the cross-tool standard, read by
  Copilot, Claude Code, and most other agents) with the pointer block below.
- **One exists already** → check if it already references CODEBASE.md. If not,
  append the pointer block to the existing file — never overwrite the rest of
  its content.

Pointer block to add:
```markdown
## Before the "read existing code" step
Always read docs/codebase/CODEBASE.md first. If it doesn't exist or is older
than 2 weeks, run the codebase-summary-kotlin-springboot skill first.
```

Do NOT repeat this check in Update mode — once the link exists it's permanent,
re-verifying it on every update wastes tokens for no benefit.

