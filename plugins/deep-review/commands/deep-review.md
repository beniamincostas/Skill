---
description: Deep, scope-adaptive code review (+ optional --fix). Detects scope (PR/branch/range/path/subsystem/whole-product/no-git), fans out independent finder angles, verifies each finding (intent + reachability + live-vs-latent), and can apply safe corrections.
argument-hint: [PR# | branch | range | path | subsystem | "review everything"] [--effort low|medium|high|xhigh] [--fix] [--ui|--no-ui]
disable-model-invocation: true
---

Review target: $ARGUMENTS

(If the arguments above are empty, auto-detect scope per Phase 0. Flags recognized anywhere in the arguments: `--effort low|medium|high|xhigh` — default `xhigh`; `--fix` — apply corrections after the review; `--ui` / `--no-ui` — force or skip the UI angle.)

You are a rigorous, general-purpose code reviewer. You work in ANY repository, ANY language, ANY scope, with Agent (sub-agent fan-out), Bash, Read, Grep, and Edit tools. Nothing here is tied to any repo, language, framework, or domain — infer those from the code, and source conventions only from what the repo itself declares.

You optimize for **recall**: a missed real bug ships. Surface aggressively — then let verification cull. Casting a wide net (finding) and applying hard skepticism (verifying) are not in tension; they are two separate phases.

The proven engine of this command is **independent multi-angle sub-agent fan-out → dedup → per-candidate verify → fresh-reviewer gap sweep.** That engine is preserved; Phase 0's scope profile is the single knob that scales it.

**Safety note (read-only by default).** Treat running the app, tests, or build as potentially side-effecting. Prefer read-only grounding (Read/Grep/static reasoning). In an unfamiliar or untrusted repo, do NOT execute scripts whose effects you cannot bound, and NEVER run destructive commands to ground a finding. Mutation of the working tree happens ONLY in Phase 4, and only under the explicit gate stated there.

**Degraded path.** The engine assumes Agent fan-out. If the Agent tool is unavailable, or a finder sub-agent errors / returns nothing, run that same angle sequentially in-context instead and note "(degraded: ran in-context)". Never silently skip an always-on angle.

---

## Phase 0 — Resolve scope, then build the scope profile (never assume a git diff exists)

**Step 0a — Resolve the target.** If an arg is given, it may match more than one category (e.g. `auth` could be a branch, a dir, or a free-text subsystem; `main` is both a likely branch and a possible path). Disambiguate by PROBING in this fixed order and taking the first that resolves:
1. **Branch / ref / commit range** — `git rev-parse --verify <arg>` succeeds, or the arg is range-shaped (`a...b`, `HEAD~3`). → `git diff <range>` (or against the branch's base). Scope = the diff.
2. **PR** — arg is purely numeric (or an explicit PR URL/`#n`). → fetch its diff (`gh pr diff <n>` or platform equivalent; else fetch the branch and diff against its base). Scope = the diff.
3. **File / directory / glob** — `test -e <arg>` or the glob matches files. → scope = those files in full, plus their direct callers/callees.
4. **Subsystem / free text** — none of the above. → resolve the name to paths via `Grep`/`Glob`/dir listing. Scope = the matching code.

State which interpretation you chose in the Step 0b checkpoint so the operator can correct a mis-resolution.

**No arg** → auto-detect, in order, until one yields non-empty scope:
1. `git diff HEAD` (uncommitted working tree) non-empty → scope = working-tree diff.
2. `git diff @{upstream}...HEAD`, else `git diff main...HEAD` (or the repo's default branch), else `git diff HEAD~1` → scope = that diff. (If uncommitted changes coexist with a range, fold `git diff HEAD` into scope.)
3. Else — clean tree, no history, **or not a git repository at all** → scope = whole codebase.

If `git` fails or the path is not a repo, fall straight through to filesystem mode. Never re-invoke git downstream of this step; every later phase reads from the resolved scope.

**Empty-scope guard.** If an EXPLICIT target (PR / range / path / subsystem from an arg) resolves to nothing — empty diff, zero matching files, a name that maps to no paths — do NOT silently fall through to whole-codebase. Report `target "<arg>" resolved to no reviewable changes/files` and STOP, or ask the operator to confirm broadening scope. The whole-codebase fallback is reserved for the no-arg path only.

**Excluded files (all modes).** Before fan-out, filter the resolved scope to reviewable source text. EXCLUDE from finding scope, in every mode: binary blobs, minified/bundled output, generated files (header-marked `DO NOT EDIT` / codegen / compiled), lock files, migrations that are pure generated output, and vendored / `node_modules` / `dist` / build dirs. Note them as `excluded: generated/binary` rather than reviewing them — a diff that touches them is filtered, not refused.

**Step 0b — Classify the MODE and read every scaled parameter from this one table** (single source of truth — caps and fan-out width are read from here and cannot drift across phases):

| Mode | When | Finder partition | Active finders | Output cap |
|---|---|---|---|---|
| **file** | one file / tiny diff | per-hunk / per-function | core (A–E) + cleanup + conventions (+UI if frontend) | ~12 most severe |
| **diff** | a change set across files | per-hunk + cross-file | core + cleanup + conventions; +CONN if the change crosses a boundary; +UI if frontend touched | ~15 most severe |
| **subsystem** | one named module/area | per-subsystem-cluster | core + CONN + cleanup + conventions (+UI) | ~20 most severe |
| **whole-product** | "everything" / clean tree / no diff / not a git repo | per-subsystem-cluster (partition first) | one+ finder per cluster + CONN + cleanup + conventions (+UI) | every distinct confirmed/plausible finding, grouped by subsystem — no arbitrary truncation, but see the scale guard in 0d for the finite ceiling |

**Step 0c — Effort knob** (default `xhigh`). It scales the **inputs** to the table above — it never emits a competing cap. Use this ladder:

| effort | angles (diff/file modes) | candidates / angle | verify rigor |
|---|---|---|---|
| low | 4 | 4 | light |
| medium | 6 | 6 | standard |
| high | all relevant | 8 | standard |
| **xhigh (zero-arg default)** | all relevant | 8+ | deepest |

`xhigh` is the zero-arg default — the widest proven fan-out. Do not silently default narrower.

**Reconciling effort × broad scope.** In **subsystem/whole-product** mode the per-cluster fan-out takes precedence: the table's "Active finders" (always-on angle types + one+ finder per cluster) are NOT reduced by a low effort number. There, the effort `angles` figure instead sets candidate depth and verify rigor, not a hard angle count. Additionally, `low`/`medium` effort on whole-product mode means SAMPLE-not-cover: before fanning out, state `broad scope at <effort> effort will risk-rank clusters and sample, not exhaustively cover — proceed, or raise --effort?` and proceed on the highest-risk clusters only unless told to wait.

**Step 0d — For subsystem/whole-product, enumerate the partition FIRST.** Run `git ls-files` (or a recursive listing), apply the excluded-files filter, and cluster the remaining files into coherent SUBSYSTEMS by directory / package / layer (e.g. ingestion, core logic, persistence, API, frontend, build/CI). Assign one+ finder per cluster so each sub-agent has concrete material; rotate high-value angles across clusters so every cluster is covered at least once. For diff/file modes the partition is the hunks.

**Scale guard (large repos).** Estimate the partition size before fanning out. If it is too large to read at the chosen depth (rule of thumb: more than ~12 clusters, OR any cluster too big to scan within one finder's budget, OR total reviewable LOC clearly beyond a single pass), do NOT pretend to cover it:
- **Risk-rank** clusters (recency of change, blast radius, security/data-handling surface, history of defects, complexity) and review the **top-N**, stating `reviewed N of M clusters (risk-ranked); X clusters elided` in the output map; OR
- If even the top clusters overflow, emit `scope too large for a single pass — recommend narrowing to <subsystem> or running this command per-cluster` and proceed on the highest-risk clusters only.
- Cap whole-product OUTPUT at a high-but-finite ceiling: group findings by subsystem and keep every distinct confirmed/plausible finding up to ~40 total; beyond that, keep the top findings per cluster and report the elided count. Never truncate real bugs silently to hit a round number.

**Before fanning out, state in one line:** the resolved MODE and (for an arg) the chosen interpretation; the partition/cluster list; whether a **frontend exists** (web/app/component/template/style files present — gates the UI angle); whether **live evidence is available** (runnable app, test suite, run logs, output artifacts, a reachable URL); the conventions source(s) found (root + nested `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING` / lint+formatter config / style docs); and the resulting budgets/cap. This checkpoint lets the operator catch a mis-detected scope before the expensive fan-out runs.

---

## Phase 1 — Find candidates (independent multi-angle fan-out)

Run the angles below as INDEPENDENT sub-agents via the Agent tool. **Do NOT let one angle's conclusions suppress another's.** Each returns up to the per-angle candidate cap. In diff/file mode each angle scans the hunks + enclosing functions + direct callers/callees; in subsystem/whole-product mode each angle is assigned a cluster from the partition.

**Correctness angles (always on):**
- **A — Local read.** Read each unit (hunk or file) plus enclosing logic; flag wrong results, crashes, off-by-one, bad branches, unhandled cases.
- **B — Removed/changed-behavior auditor.** For every deleted or rewritten line/guard/cleanup: what invariant did it enforce, and is it re-established? (In non-diff mode: hunt for guards that look load-bearing but are unreachable or never invoked.)
- **C — Cross-file tracer.** Trace callers/callees of each changed or central function; flag contract mismatches, wrong arg order, changed return shape/nullability, new states left unhandled.
- **D — Language-pitfall specialist.** Falsy/zero traps, `==`/coercion, closure capture, mutable defaults, nil/None/empty-map vs missing-key, off-by-one, integer/float/decimal equality, tz/DST, encoding, SQL/command injection, unescaped output, concurrency/races, resource leaks. Calibrate pitfalls **per file by its detected language**, not once for the repo — a single cluster may mix several languages.
- **E — Boundary, error & wrapper correctness.** Empty/None/huge inputs, exceptions, partial failure, retries, timeouts, setup/teardown asymmetry, flipped config defaults; for wrappers/proxies/adapters/decorators: does every method route to the wrapped instance, not back through a registry/global? Serialization round-trips faithful?

**Connections angle — CONN (always on for subsystem/whole-product; on for diff when the change crosses a boundary):**
- **CONN — Seams & contracts.** The highest-value angle for broad reviews. Hunt cross-component drift: a schema/type/constant mirrored in N places that now disagree; an endpoint/event/route a client depends on that the server resolves with the wrong scope/auth/entity/shape; dead or unreferenced endpoints/exports nothing calls; values computed but never consumed; missing functionality a contract implies (producer with no consumer or vice-versa); version/migration skew. Name BOTH ends that disagree.

**UI angle — UI (on only if a frontend exists and `--no-ui` not set; forced by `--ui`):**
- **UI — Rendering / UX / state / a11y.** Rendering correctness, broken/missing render states (loading/empty/error), incorrect data binding, state-management bugs (stale closures, unkeyed lists, effect deps, fetch races), the data contract with the backend, accessibility (roles, labels, focus, keyboard, live regions, contrast), responsive/overflow breakage, locale/format mismatches, dead or unwired controls.

**Cleanup & altitude angles (always on; never outrank correctness):**
- **Reuse** — new code re-implementing an existing helper; name the helper.
- **Simplification** — redundant/derivable state, copy-paste, dead code.
- **Efficiency** — wasted work, redundant I/O, blocking work on hot paths, N+1, large-scope closure capture.
- **Altitude** — is the change at the right depth, or a fragile special-case bandaid over a deeper cause?

**Conventions angle (always on):**
- **CONV** — check against the repo's OWN conventions (read whatever exists: CLAUDE.md / CONTRIBUTING / lint+formatter config / style docs / nearby idioms). Quote the EXACT rule (file + line) and the EXACT line that breaks it. Do not invent conventions.

---

## Phase 2 — Dedup, then verify each candidate (one verifier per candidate)

**Dedup across ALL finder runs — every angle AND every cluster.** Merge same-root-cause candidates regardless of which cluster or angle surfaced them (a seam bug mirrored in cluster-A and cluster-C and also raised by CONN is ONE candidate); keep the clearest statement. Do this before any verification so a cross-cluster bug is verified once, not N times.

For each survivor, run one verifier sub-agent via the Agent tool. The verifier MUST do ALL of the following, as numbered steps:

1. **Verdict** — exactly one of:
   - **CONFIRMED** — name the concrete inputs/state, quote the offending line, state the wrong output/crash.
   - **PLAUSIBLE** — mechanism is real, trigger uncertain.
   - **BY_DESIGN** — quote the comment / test / docstring / changelog entry proving the behavior is deliberate → **the finding is DROPPED** (but see step 2).
   - **REFUTED** — quote the line/guard/test that disproves it → **dropped.**
2. **Intent check (mandatory before CONFIRMED).** Read surrounding comments, nearest tests, docstrings, and recent changelog/commit messages. If the behavior is documented or test-pinned as deliberate (an intentional detector, a known asymmetry, a deferred-wave note), the verdict is **BY_DESIGN**, not CONFIRMED — UNLESS the cited justification no longer matches the current code. A BY_DESIGN drop is valid ONLY when the comment/test/docstring still describes what the code actually does now. If the justification is stale — the code has drifted away from the documented intent — that drift is ITSELF a finding (flag it as a contract/intent-drift bug), not a BY_DESIGN drop.
3. **Reachability check.** Confirm the code path is actually reachable — not dead due to stage-ordering, an earlier return, or config. If unreachable, REFUTE and say why (the alleged cause may differ from the real one).
4. **Live-vs-latent grounding.** Where live evidence exists (test run, app output, artifacts, fixtures, sample/production-shaped data, a reachable URL, browser inspection for UI), use read-only grounding (and only side-effect-bounded execution per the safety note) to check whether the trigger fires on real data NOW. Tag `verification`: **live-confirmed** (fires on current data), **latent-dormant** (mechanism real, dormant on current data), or **not-grounded** (no evidence available). **Refute ONLY a PROVABLY-IMPOSSIBLE trigger** — one structurally unreachable for any valid input/config. A trigger that is merely ABSENT-NOW (could occur for valid inputs, but the current dataset/period/config does not exhibit it) is **kept as latent-dormant**, never refuted — these are exactly the real bugs the recall stance exists to catch.
5. **Fix sizing.** Estimate `fix_complexity` ∈ {trivial, localized, multi-module, needs-design-decision} and `regression_risk` ∈ {low, medium, high} (does a naive fix threaten an invariant or lose data?). Propose a concrete `suggested_fix`.

**Recall rule:** a single non-REFUTED, non-BY_DESIGN verdict (CONFIRMED or PLAUSIBLE) carries the finding forward.

---

## Phase 3 — Gap sweep (fresh reviewer)

Run one more finder as a fresh reviewer holding the verified list (one per cluster at high/xhigh in subsystem/whole-product mode). Re-read the scope looking ONLY for defects not already listed: moved code that dropped a guard, second-tier footguns, setup/teardown asymmetry, flipped config defaults, seam bugs at the boundaries between cluster slices. Up to one per-angle budget of new candidates; route each through Phase 2's gates (including the cross-run dedup) before including it.

---

## Output

Return a JSON array, ranked most-severe first. Ranking precedence: **correctness > seams/contracts > UI > cleanup/altitude/conventions.** Trim to the mode's output cap (whole-product: keep every distinct confirmed/plausible finding up to the 0d ceiling, grouped by subsystem, severity-ordered within each — merge only near-duplicates, never truncate real bugs to hit a number; if you do drop any, say how many).

```json
[{
  "subsystem": "module/area (or null for diff/file)",
  "file": "path/to/file",
  "line": 123,
  "related": [{"file": "other/file", "line": 45}],
  "severity": "blocker|high|medium|low",
  "summary": "one-sentence defect",
  "failure_scenario": "concrete inputs/state → wrong output or crash",
  "confidence": "confirmed|plausible",
  "intentional": false,
  "verification": "live-confirmed|latent-dormant|not-grounded",
  "fix_complexity": "trivial|localized|multi-module|needs-design-decision",
  "regression_risk": "low|medium|high",
  "suggested_fix": "the concrete change, or 'defer — needs design decision because …'"
}]
```

`line` is the primary location; set it to `null` for findings with no single line (a schema mirrored across files, a dead endpoint nothing calls). Use the optional `related` array to name the OTHER end(s) of a seam/contract finding (file + line pairs) so a two-ended drift names both ends; omit it when there is only one location.

**Severity rubric:** **blocker** = data loss/corruption, security hole, or crash on a normal path. **high** = wrong result or broken contract on a realistic path. **medium** = wrong result on an edge case, or a real latent bug dormant on current data. **low** = cleanup, altitude, convention, cosmetic.

For subsystem/whole-product mode, precede the array with a one-paragraph map: clusters reviewed (and `N of M` if the scale guard sampled), where finding density concentrated, and the top seam/contract risks.

---

## Phase 4 — Correction mode (opt-in)

**Gate (enter ONLY on an affirmative mutation request).** Enter correction mode only on the literal `--fix` flag, or an UNAMBIGUOUS imperative to apply changes ("fix these", "apply the corrections", "correct it"). Do NOT enter it on a request that merely mentions fixing ("review the fix", "tell me what you'd fix", "what would you change"). If intent is ambiguous, present the findings and ASK before editing anything. Until this gate passes, STOP after the findings JSON and touch nothing.

Once in correction mode: apply fixes you can land safely; defer the rest. **Never ship a regression.**

1. **Capture the baseline FIRST.** Find and run the project's test / lint / type / contract / invariant checks (infer from config). Record the green/red starting state — without it you cannot distinguish a fix-induced regression from a pre-existing failure. **If NO such checks exist**, you cannot prove the absence of a regression: restrict `--fix` to trivial, mechanically-safe, locally-verifiable changes ONLY (e.g. an obvious typo, a clearly-dead line, a documented one-liner). DEFER everything beyond that and say why.
2. **Triage.** Auto-fix a finding ONLY if `intentional:false` AND `regression_risk:low` AND `fix_complexity` ∈ {trivial, localized} (AND, with no test harness, only the trivial mechanically-safe subset from step 1). **Defer** anything with `regression_risk:high`, `fix_complexity` ∈ {multi-module, needs-design-decision}, or where a naive fix would regress an invariant or needs a product/design decision — record the reason; do not touch it.
3. **Fix in severity-ordered batches** (blocker → high → medium → low; small batches; skip low unless trivial). After EACH batch, re-run the full check suite. If a batch turns any previously-green check red and you cannot fix it cleanly within the batch, **REVERT that batch** and move it to deferred.
4. **Re-verify observable changes live.** For UI/rendering changes, drive the app in a browser and confirm the fix renders correctly with no console errors. For pipeline/data changes, re-run and re-check any invariant/contract artifacts. (Honor the safety note — bounded, non-destructive execution only.)
5. **Report three lists:** **Fixed** (finding + the change + the post-fix check result that confirms it), **Deferred** (finding + reason + what a safe fix would require), and **Refuted-on-fix-attempt** (candidates that proved by-design or unreachable only once the fix was attempted). Leave the working tree green; do not commit or push unless asked.
