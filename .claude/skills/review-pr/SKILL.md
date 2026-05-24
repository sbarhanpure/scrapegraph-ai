---
name: review-pr
description: "Tiered, evidence-based PR review covering Operational Excellence + Engineering Excellence + project-specific hard rules. Sections are gated so small PRs get focused reviews and large PRs get the full battery. Usage: /review-pr <PR#> [--repo owner/name]"
argument-hint: "<PR#> [--repo owner/name]"
---

You are a PR reviewer. Your job is evidence-based, not speculative.

**Four rules that override everything else:**

1. Every non-N/A finding must cite its source (file:line, command output, or diff line). No evidence = no claim.
2. State explicitly what you could NOT check. Never imply completeness you do not have.
3. Sections are tier-gated. A section whose gate is not met is NOT run and NOT emitted as an N/A table. It IS listed under "What was NOT checked" so the reader knows it was considered.
4. Noise dilutes signal. A 50-row N/A table is not "comprehensive review" — it's filler that buries real findings. Be ruthless about gating.

The review covers three layers:

- **OE — Operational Excellence:** observability, resilience, deployability, runbooks. Does this run well in production?
- **EE — Engineering Excellence:** correctness, design fit, scope discipline, readability, test adequacy, supply chain. Is this engineered well as code?
- **Project hard rules:** mechanical, repo-specific blockers (see Tier 3).

---

## Parse Arguments

Arguments: `$ARGUMENTS`

- First positional arg: PR number (required)
- `--repo owner/name`: overrides default repo (default: infer from `git remote get-url origin`)

If no PR number is provided, output: `Usage: /review-pr <PR#> [--repo owner/name]` and stop.

---

## Step 1 — Identify Repo

```bash
git remote get-url origin 2>/dev/null
```

Strip the GitHub URL prefix to get `owner/repo`. Use `--repo` value if passed.

---

## Step 2 — Fetch PR Metadata and Check Mergeability First

```bash
gh pr view <PR#> --repo <owner/repo> \
  --json title,body,headRefName,baseRefName,author,additions,deletions,changedFiles,\
mergeable,mergeStateStatus,statusCheckRollup,files,commits
```

**Before reading anything else, check `mergeable` and `mergeStateStatus`:**

- If `mergeable = CONFLICTING` or `mergeStateStatus = DIRTY`: post the blocking comment below and stop. A conflicting branch cannot be safely assessed. CI can be green while the branch conflicts; the merge button block is the evidence.

```bash
gh pr review <PR#> --repo <owner/repo> --comment --body "$(cat <<'BLOCK'
## Claude Code PR Review

🔴 **Merge conflict — review blocked**

Branch has unresolved conflicts with the base branch. Resolve and re-push before review continues.

Detected via \`mergeable: CONFLICTING\` in the GitHub API. CI status checks can be green while the branch still conflicts — that is why the merge button is blocked even when all checks pass.

*Reviewed by Claude Code /review-pr*
BLOCK
)"
```

Record from `statusCheckRollup`: name and conclusion of every check. Cite these in the output.

---

## Step 3 — Read the Full Diff (No Silent Truncation)

```bash
FULL_DIFF=$(gh pr diff <PR#> --repo <owner/repo>)
LINE_COUNT=$(echo "$FULL_DIFF" | wc -l | tr -d ' ')
echo "Diff line count: $LINE_COUNT"
```

- **≤ 800 lines:** use `$FULL_DIFF` directly.
- **> 800 lines:** do not truncate silently. Get the changed file list and read each file's diff after checking out the branch (Step 7):

  ```bash
  gh pr view <PR#> --repo <owner/repo> --json files --jq '[.files[].path]'
  git diff origin/<baseRefName>...<headRefName> -- <file>
  ```

  If you still cannot read a file, name it in "What Was NOT Checked."

---

## Step 4 — Triage and Tier Selection

Before any checklist runs, classify the PR. This drives which sections fire.

### 4.1 — PR class (drives which Tier-2 sections fire)

| Class | Trigger (file paths in diff) | Tiers that fire |
|---|---|---|
| **Doc-only** | Only `*.md`, `docs/**`, `.github/ISSUE_TEMPLATE/**`, `CHANGELOG.md` | T-0, T-1 (readability + PR-description accuracy only), T-3 (rules that apply to docs) |
| **Config-only** | Only `.github/workflows/**`, `.env.example`, `package.json` deps, `tsconfig*`, `biome*`, `.gitignore` | T-0, T-1 (all), T-2.5 deployment + T-2.6 runbooks + T-2 supply chain, T-3 |
| **Migration-only** | Only `supabase/migrations/**` | T-0, T-1 (all), T-2.3 API/schema + T-2.4 migration + T-2.1 observability if user-scoped, T-3 |
| **Code change** | Any `bot/**`, `scripts/**`, `tests/**` | All tiers, all applicable subsections |
| **Mixed** | Any combination | Union of all applicable tiers/subsections |

Record the class. State it in the output triage line.

### 4.2 — Size class (drives review depth and a "consider splitting" flag)

| Class | Diff lines | Approach |
|---|---|---|
| Small | ≤ 200 | Read entire diff inline; check every changed line |
| Medium | 200–800 | Read entire diff inline; spot-check generated/repetitive sections |
| Large | > 800 | Per-file diffs after branch checkout; explicit listing of any file skipped |

DORA elite-performer research correlates PRs > ~400 lines with longer review time and higher defect rate. **If Large, surface this in the output**: "PR is large — recommend splitting on next iteration." This is feedback, not a block.

### 4.3 — AI-authored flag (drives Tier-1.6 LLM-authorship drift checks)

The PR is AI-authored if any of:

- Commit author / `Co-Authored-By` contains `claude`, `gpt`, `bot`, `aider`, `cursor`, `copilot`
- Branch name starts with `claude/`, `bot/`, `ai/`, `cursor/`
- PR body contains "🤖 Generated with" or equivalent automation marker

If AI-authored → Tier-1.6 fires. State the flag on the output triage line.

---

## Step 5 — Load Context on Demand

Read only what the diff requires. Do not preload.

| If the PR touches | Read |
|---|---|
| Any file (always) | Repo's `CLAUDE.md` — §0 build-status table, §16 hard rules, §16 mechanical enforcement table |
| `bot/claude/prompt.ts` or `bot/tools/*.ts` | `~/projects/ai-principles/quick-reference.md` |
| `SYSTEM_PROMPT` body changed | `~/projects/ai-principles/principles/07-testing.md` §4–7 |
| `supabase/migrations/` changed | The migration file directly |
| `.github/workflows/` changed | The modified workflow file directly |
| Public API surface changed (new exported function, new route, new tool) | The callers — `grep -rn "<export-name>"` to verify no orphans |
| ADR-numbered citation in diff | The cited ADR file — verify it actually says what the diff claims |

---

## Step 6 — Apply Checklist (Tiered)

Score: ✅ pass / ⚠️ concern / 🔴 block / N/A. **Every non-N/A score needs evidence.** **Skip an entire section** if its gate is not met — record under "What was NOT checked." Never emit an empty N/A table.

---

### TIER 0 — Triage (every PR)

| Check | Status | Evidence |
|---|---|---|
| Targets a feature branch, not `main` → `main` | | `headRefName` / `baseRefName` from Step 2 |
| `mergeable` not CONFLICTING | | from Step 2 |
| CI checks — name + conclusion for each | | from `statusCheckRollup` |
| Size class (Small/Medium/Large) — flag Large for splitting | | from Step 4.2 |
| PR class detected | | from Step 4.1 |
| AI-authored flag | | from Step 4.3 |
| Title matches scope — diff stays on topic | | scan changed files vs. title |

---

### TIER 1 — Universal Review

Fires on every PR. For Doc-only PRs only §1.2 and §1.7 fire.

#### 1.1 Correctness & scope discipline (EE)

Gate: any code change.

| Check | Status | Evidence |
|---|---|---|
| Code does what the title/description says — no unrelated edits, no surprise refactor, no mass rename | | list any out-of-scope files |
| No half-finished implementations — stubs, `throw new Error("not implemented")`, unreferenced new exports | | grep diff for `TODO`, `FIXME`, `not implemented`, unreferenced exports |
| No backwards-compat shims for code that hasn't shipped yet — fresh code should be straightforward, not pre-bridged | | inspect new wrappers / aliases |
| No design-for-hypotheticals — new abstractions have ≥ 2 real callers in the same PR | | grep callers of new abstractions |

#### 1.2 Readability (EE)

Gate: any change.

| Check | Status | Evidence |
|---|---|---|
| New names are descriptive; renamed exports have callsite updates in the same PR | | grep callsites of renamed exports |
| New comments explain WHY (non-obvious constraint, subtle invariant, workaround). Flag commentary that just narrates the code or names the calling flow | | inspect new `//` and block comments |
| No dead code — commented-out blocks, `// removed` markers, orphan exports | | grep diff for `// removed`, commented-out code blocks |

#### 1.3 Test adequacy (EE) — comprehensive testing for code changes

Gate: any code change (skip for doc/config-only).

This section is split into four sub-checks: **unit**, **integration / contract**, **regression**, and **meta**. Run all four for any code-containing PR. The goal is test STRENGTH, not test count.

**Mutation-adequacy lens (applies to all four):** if you deleted the function body and returned `null`, would ANY test fail? If only spy/call-count assertions exist (`toHaveBeenCalled()` without asserting return value or side effect), the function is untested in the meaningful sense — flag ⚠️.

##### 1.3.a — Unit tests (every new/changed function)

| Check | Status | Evidence |
|---|---|---|
| Each new exported function has at least one unit test — failing if it returned `null` or garbage (per-export coverage) | | cite test name or flag gap |
| Happy path covered — typical input produces expected output | | |
| Error / failure paths — for each `throw` or error return in new code, a test exercises it | | cite test name or flag gap |
| Boundary conditions — for any cap / clamp / min / max, the boundary value AND boundary±1 are covered | | |
| Negative-input rejection — for each input validation, a test verifies invalid input is rejected (not just that valid input is accepted) | | cite test or flag gap |
| Tests assert on behaviour (output value, shape, side effects), not implementation details that could change without breaking the feature | | |
| Test data realism — flag shortcuts (`userId = "test"`, all fields `null`) that would miss type coercion or constraint violations | | |

##### 1.3.b — Integration / contract tests (every new module boundary)

| Check | Status | Evidence |
|---|---|---|
| Any new external boundary (DB query, HTTP call, SDK call, tool invocation) has a test that exercises the integration shape — not just the function in isolation | | cite test or flag gap |
| Tool output schema changes have a matching `tests/tools/contract-shape.test.ts` update so the contract is enforced | | |
| Cross-module flows (handler → service → DB) have at least one test that runs the flow end-to-end at the module-integration level | | |
| Concurrent-access test for any read-modify-write or atomic operation (e.g. `DELETE...RETURNING` replay guard) — two simultaneous calls, only one succeeds | | N/A if no atomic ops |
| No hand-authored fake `messages.create` / Anthropic SDK responses — these give false confidence; integration tests must hit real adapters or properly-recorded fixtures | | grep `mock.*messages.create` |

##### 1.3.c — Regression tests (every bug fix and every behavior change)

| Check | Status | Evidence |
|---|---|---|
| **Bug-fix PRs:** the failing scenario from the bug has a test that would FAIL on the parent commit and PASS on this commit. Test name references the bug (e.g. `regression: handles X correctly when Y`) | | cite the regression test or flag missing |
| **Behavior-change PRs:** the test that asserted the OLD behavior is updated (not deleted) — or a comment explains why the old behavior is no longer covered | | |
| Quality-gate scripts have a Layer 3 real-directory regression guard — a test that scans the actual `scripts/` or `bot/` directory and asserts zero violations | | N/A if no gate scripts |
| Reference-question eval (`tests/reference-questions.json`) still passes — if `bot/claude/` or `bot/tools/` touched, the eval is the regression suite for AI behavior | | cite per-question results in PR body or 🔴 if missing per `07-testing.md §4.1` |
| Snapshot / golden-file tests, if any, are updated with the diff inspected — not blindly accepted | | inspect any `__snapshots__` diff |
| For schema migrations: a test exists that asserts the post-migration query still returns the shape callers expect (regression guard against silent breakage) | | N/A if no migrations |

##### 1.3.d — Test count & meta

| Check | Status | Evidence |
|---|---|---|
| Test count cross-check: if PR claims "+N tests," count new `it(` / `test(` / `it.skipIf(` in diff — state actual (±2 acceptable for parameterised `it.each`) | | stated: N / actual: N |
| New tests are picked up by the runner — not orphaned outside the test glob, not `it.skip` / `describe.skip` left behind | | grep diff for `.skip(` |
| If `npm run test` is run locally (Step 7): the new tests appear in the runner output, not silently filtered out | | command output |
| If the PR removes tests, the removal is justified in the PR description (test was duplicative, behavior was deleted, etc.) — silent test deletion is ⚠️ | | grep diff for `-  it(` and `-  test(` |

#### 1.4 Security baseline (OWASP-aligned, EE+OE)

Gate: any code change.

| Check | Status | Evidence |
|---|---|---|
| **A02/A05 — Secrets in diff.** No `SERVICE_ROLE`, `_SECRET`, `_KEY`, token literals, base64 blobs, `.env` contents | | grep |
| **A03 — Injection.** Any new SQL / shell / HTTP / HTML construction is parameterized or properly escaped, never string-concatenated | | inspect query builders, `exec`, template strings touching `${userInput}` |
| **A01 — Broken access control.** Any new endpoint / handler / tool runs an auth check before data access; user-scoped tools use `getUserSupabase(userId)`, never service-role | | cite `check:tenant-scoping` CI, or inspect handler |
| **A09 — Logging failures.** No credentials/tokens in `console.log`; no full `ctx` / Telegram-object dumps containing PII | | grep `console\.log.*(token|key|secret|password|credential)`; grep `console\.log\(ctx\)` |
| **A10 — SSRF.** New outbound URL / file path / fetch target is validated against an allowlist if user-influenced | | N/A if no user-influenced outbound calls |
| User input clamped — enum allowlists, numeric min/max — before reaching any query | | |

#### 1.5 Supply chain (EE+OE)

Gate: `package.json` changed, or any new `import` line in diff.

| Check | Status | Evidence |
|---|---|---|
| `package.json` deps changed → `package-lock.json` updated in same PR | | |
| **Hallucinated-package check.** Every new bare-name `import` in diff resolves to a package in `package.json` or a real relative path. (Recent research: ~20% of LLM-suggested packages don't exist; ~58% of those are repeats — slop-squatting attack surface.) | | grep `^[+].*import .* from ['"][^./]` in diff; cross-check against `package.json` |
| `npm audit` clean, OR known issues documented in PR body with rationale | | command output |
| New dep license is compatible with project license | | check `npm view <pkg> license` for any new dep |

#### 1.6 LLM-authorship drift (gate: AI-authored flag = true from Step 4.3)

LLM-authored PRs have specific failure modes distinct from runtime AI drift. These rows catch authorship issues.

| Check | Status | Evidence |
|---|---|---|
| Diff scope matches title — flag if unrelated files were edited "while we were in there." LLMs scope-creep silently | | list files outside the stated scope |
| Hallucinated-package check — see §1.5 (this is the #1 LLM authorship failure mode) | | |
| Each new exported function has at least one test for null / empty / error input — LLMs systematically write only the happy path | | cite test or flag gap |
| New raw query construction uses parameterized API, not string concat — LLMs default to concatenation | | inspect query builders |
| No fabricated citations — links to docs / ADRs / files that don't exist, library-behavior claims that contradict the package README | | spot-check 1-2 cited URLs / library claims in diff and PR body |
| Comments don't reference "this fix" / "added for issue #123" / "the X flow" — author/task context belongs in PR description, not in code | | grep new comments |

#### 1.7 PR description accuracy (EE)

Gate: every PR.

The PR description is a contract. Cross-check stated claims against evidence.

**Deferral check** — grep the PR description for "follow-up", "future PR", "handle later", "separate PR". For each match: is a GitHub issue URL or a named plan document / ADR cited in the same sentence or paragraph? No link and no plan-doc citation = ⚠️.

| Claim | Stated in PR | Actual | Status |
|---|---|---|---|
| Test count ("+N tests") | | count `it(` / `test(` in diff | |
| File LoC for new files ("N LoC") | | `wc -l <file>` | |
| CI green | | `statusCheckRollup` JSON | |
| Changed file list matches | | `gh pr view --json files` | |
| Deferred functionality | "follow-up PR" / "future PR" / "handled later" language | GitHub issue or named plan-doc/ADR linked for each deferral? | |
| Cited files/ADRs exist | references in description | verify each cited path exists in repo | |

---

### TIER 2 — Risk-Gated

Each subsection has an explicit gate. **Skip entirely** if the gate is not met — note in "What was NOT checked."

#### 2.1 Observability (OE)

Gate: new I/O, new handler, new background job / cron, new external call, new error path on user-visible operation.

| Check | Status | Evidence |
|---|---|---|
| New code path emits structured logs with enough context to debug from logs alone (no "something failed") | | |
| Error paths log at appropriate level — `error` for unrecoverable, `warn` for recoverable. Don't `info`-log everything | | |
| Per-call timing/counter for new external dependency (Strava / Oura / Anthropic / Supabase) so latency regressions are visible | | |
| User-scoped operations include `userId` in log context (multi-tenant debugging) | | |
| New failure mode has a way to be alerted on (counter, log query, or runbook entry) | | |

#### 2.2 Resilience (OE)

Gate: new external call, new write path, new atomic operation, new background job.

| Check | Status | Evidence |
|---|---|---|
| External call has an explicit timeout — no unbounded `fetch` / SDK call | | |
| Retry policy is bounded (capped attempts, exponential backoff) and only fires on idempotent operations | | |
| Partial-write failure does not corrupt state — wrapped in a transaction, or compensated, or marked recoverable | | |
| Blast radius bounded — one user's failure does not affect others; one bad row does not crash a batch | | |
| Cron / background job is idempotent on re-run — safe to fire twice | | |

#### 2.3 API & schema compatibility (EE+OE)

Gate: public exports changed, migrations, tool interfaces.

| Check | Status | Evidence |
|---|---|---|
| Renamed exports keep the old name exported alongside the new for one release cycle | | |
| Removed exports have zero remaining callsites in the repo — `grep` confirms | | |
| Migration is additive (new columns nullable, new tables new). Destructive change has an ADR and a documented rollout | | |
| Tool output schema change has matching `tests/tools/contract-shape.test.ts` update | | |
| Log format changes — if any downstream parses logs, flag the format change | | |

#### 2.4 Migration safety (OE)

Gate: `supabase/migrations/` changed.

| Check | Status | Evidence |
|---|---|---|
| Rollback script exists at `supabase/migrations/rollbacks/NNN_rollback.sql` | | |
| Rollback is idempotent (can run twice without error) | | read rollback file |
| No `NOT NULL` column added without a `DEFAULT` before the code that supplies the value ships | | read migration; confirm app code is in same PR or DEFAULT is dropped in a follow-up |
| No new `UNIQUE` constraint on data that might already have duplicates — PR description confirms pre-check, or constraint is on a new column only | | |
| Every table with `CREATE POLICY` has `GRANT ... TO authenticated` in the same migration | | cite `check:migration-grants` CI result |
| If `user_id` column added to a user-scoped table: backfill uses correct sentinel UUID; default is dropped in same migration after backfill | | |
| PR description states how many rows were affected by any data backfill | | |

#### 2.5 Deployment safety (OE)

Gate: new env vars, new workflows, `package.json` lifecycle hooks, new infra config.

| Check | Status | Evidence |
|---|---|---|
| New env vars are in `.env.example` | | N/A if no new vars |
| Startup assertion in `bot/index.ts` fails loudly — names the missing var — if any new required var is absent | | N/A if no new vars |
| PR description has a "Deployment checklist" naming each new var and where to obtain its value | | N/A if no new vars |
| New cron workflow has `concurrency:` set to prevent overlapping runs | | N/A if no new workflows |
| `package.json` lifecycle hooks (`prepare`/`postinstall`/`preinstall`) guard against missing `.git` with `[ ! -d .git ] && exit 0` | | N/A if hooks unchanged |
| Railway rollback feasible — reverting to previous deployment restores correct behaviour. Note if a data migration or new required env var makes rollback non-trivial | | |
| Change is feature-flagged OR independently revertible without a follow-up PR — bonus: kill switch documented | | inspect for `if (flag)` guards or rollback path |

#### 2.6 Runbook / docs / ADR impact (OE+EE)

Gate: operator-visible behavior changed, new alert, new failure mode, new external dep, new architectural decision.

| Check | Status | Evidence |
|---|---|---|
| `README.md` updated if developer-facing behavior changed | | |
| `docs/runbooks/` updated if operator-facing behavior changed (new alert, new failure mode, new manual recovery step) | | |
| ADR added under `docs/architecture/decisions/` if a non-trivial architectural decision was made (new external dep, new data model, new layer crossing) | | |
| `CLAUDE.md` updated if hard rules / mechanical-enforcement table / build-status table changed | | |
| API / tool reference docs updated when public surface changed | | |

#### 2.7 AI runtime drift (EE+OE)

Gate: `bot/claude/`, `bot/tools/`, or `tests/reference-questions.json` touched. This catches drift in the running AI system (distinct from §1.6, which catches drift in the AI that authored the PR).

**Model ID** — if `bot/claude/prompt.ts` changed:

- `CLAUDE_MODEL` is a valid ID. **Do NOT rely on this skill file for the valid set** — check `tests/unit/model-ids.test.ts` directly; that file is the authoritative list. Grep the diff for the constant, then verify it appears in `VALID_MODEL_IDS`.
- `tests/unit/model-ids.test.ts` covers the new ID if it changed.

**System prompt** — if the body of `SYSTEM_PROMPT` changed:

- Was the prompt-change checklist followed? (9 items, `quick-reference.md` "Prompt Change Checklist")
- Does the PR description include per-question results from `tests/reference-questions.json`? If not: 🔴 — per `07-testing.md §4.1`, no prompt change merges without reference-question results.
- Cost regression: median token counts within +20% of the previous merged run? Cite from PR or `ai_request_log`.

**Tool output schema** — if any `bot/tools/*.ts` interface changed:

- Field names/types/nullability changed? Is `tests/tools/contract-shape.test.ts` updated in the same PR?
- Renamed field? Old name still exported alongside the new name for one release cycle?

**Token budget & history** — if `bot/claude/chat.ts` or `bot/claude/tools.ts` changed:

- `max_tokens` unchanged? If raised, was tool output reduced to compensate?
- Conversation history cap still ≤ 20 rows (10 message pairs)?
- Input token budget per turn still ≤ 8,000?

**Tool output bounds** — if any `bot/tools/*.ts` changed:

- New outputs row-capped (≤ 50 default, ≤ 200 absolute)?
- Returned values pre-formatted at the tool boundary (units converted; Claude never converts)?
- Optional fields handled in the system prompt (or does Claude receive silent nulls)?

---

### TIER 3 — Project-specific hard rules (every code-containing PR)

These are mechanical, repo-specific blockers. Fire on any code-containing PR.

| Rule | Status | Evidence |
|---|---|---|
| No direct push to `main` — PR targets a feature branch, not main-to-main | | |
| Branch up to date — `mergeable` is not CONFLICTING (from Step 2) | | |
| No `.env` secrets committed — grep diff for `SERVICE_ROLE`, `_SECRET`, `_KEY`, raw token patterns | | |
| No `execute_sql` tool added anywhere in the diff | | |
| No source file > 600 lines — check files that were added or substantially expanded (`wc -l`) | | |
| Every new `bot/tools/*.ts` has a matching test in `tests/tools/` | | N/A if no tool files changed |
| Every new migration has a rollback in `supabase/migrations/rollbacks/` | | N/A if no migrations |
| `CLAUDE.md` §0 build-status table updated for any added, removed, or status-changed file | | N/A if no new files |
| No `claude-3-*` IDs and no bare family names without version suffix (e.g. `claude-sonnet-4`) in changed files — grep diff | | |
| New `scripts/*.ts` that import `bot/lib/supabase.js` call `runScript(main, import.meta.url)` | | cite `check:script-entrypoints` CI result, or N/A |

---

## Step 7 — Run Local Checks

Gate: code-containing PR (Code change or Mixed class) AND repo is cloned under `~/projects/`. Skip for Doc-only / Config-only; cite CI instead.

```bash
git fetch origin
git checkout <headRefName>
git pull origin <headRefName>

REVIEW_SHA=$(git rev-parse HEAD)
echo "Reviewing SHA: $REVIEW_SHA"

# Verify node_modules match the lockfile; note if npm install was needed
npm ls --depth=0 2>&1 | grep -E "^(MISSING|ERR)" | head -5

npm run lint  2>&1 | tail -10
npm run build 2>&1 | tail -10
npm run test  2>&1 | tail -10
# Also run any check: scripts this PR adds or modifies
```

**Before reporting results, verify:**

- `git rev-parse HEAD` matches the branch tip SHA shown on GitHub.
- If you had to run `npm install` to resolve missing packages, say so — CI runs `npm ci` from scratch and may surface issues your local run masked.

If repo is not cloned locally: skip Step 7 entirely and state it in "What was NOT checked."

---

## Step 8 — Post Review Comment

Use the structure below. **Only emit a section's table if the section actually fired.** List skipped sections in "What was NOT checked" with the reason.

```bash
gh pr review <PR#> --repo <owner/repo> --comment --body "$(cat <<'REVIEW'
## Claude Code PR Review

**Branch:** \`<headRefName>\` → \`<baseRefName>\`
**Scope:** <changedFiles> files, +<additions> / -<deletions> lines
**Class:** <Doc-only | Config-only | Migration-only | Code change | Mixed>
**Size:** <Small | Medium | Large> <append "— recommend splitting" if Large>
**AI-authored:** <yes | no>
**Reviewed at SHA:** \`<REVIEW_SHA>\` (or "not checked out locally")
**CI status:** <each check name + conclusion from statusCheckRollup>
**Tiers fired:** T-0; T-1 (<subsections>); T-2 (<subsections>); T-3

---

### TIER 0 — Triage
| Check | Status | Evidence |
|---|---|---|
| ... | | |

### TIER 1 — Universal
[Emit only the §1.x subsections whose gate fired. Each as its own table.]

### TIER 2 — Risk-gated
[Emit only the §2.x subsections whose gate fired. Each as its own table.]

### TIER 3 — Project hard rules
| Rule | Status | Evidence |
|---|---|---|
| ... | | |

---

### Local check results
**SHA reviewed:** \`<REVIEW_SHA>\` (or "not checked out locally")
**node_modules state:** <clean | required npm install — note>
\`\`\`
npm run lint  → <result>
npm run build → <result>
npm run test  → <N passed | N skipped>
\`\`\`

---

### What was NOT checked

[Always list, with the reason for each:]
- Tier sections that did NOT fire because their gate wasn't met (e.g. "T-2.4 Migration safety: skipped — no \`supabase/migrations/\` changes")
- Behavioural test adequacy — test existence/count verified; whether tests catch production regressions is a human call
- Railway dashboard state (env vars, deployment history) — requires Railway access
- Reference-question set execution — only verifiable if PR includes per-question results
- Data migration correctness for existing rows — requires staging DB with prod-representative data
- Any files skipped due to diff size — name them explicitly

---

### Summary
<2–3 sentences. Overall verdict and the single most important unresolved issue. Name any 🔴 explicitly. If size is Large, append a one-line "Recommend splitting future PRs of this scope" — feedback, not a block.>

*Reviewed by Claude Code /review-pr*
REVIEW
)"
```

### Verdict rules

- Any 🔴 → **"Do not merge until blocking issues are resolved."**
- Only ⚠️ → **"Safe to merge after addressing concerns."**
- All ✅ → **"Looks good to merge."**
- Size class Large → append: **"Recommend splitting future PRs of this scope."** (feedback line, does not change the merge verdict)
