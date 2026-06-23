# deep-review

A **scope-adaptive code-review (and optional auto-fix)** slash command for
[Claude Code](https://claude.com/claude-code). General-purpose — no repo,
language, framework, or topic baked in.

It detects what you're reviewing (a PR, a branch, a commit range, a path, a
named subsystem, or the whole product — even in a non-git tree), fans out
independent finder angles as sub-agents, verifies every candidate (intent +
reachability + live-vs-latent grounding before confirming), sweeps for gaps,
and emits ranked findings with rich metadata. With `--fix` it then applies the
safe corrections in severity-ordered, test-gated batches and defers the risky
ones — never shipping a regression.

## Install

**Personal (all your projects):**
```bash
mkdir -p ~/.claude/commands
curl -fsSL https://raw.githubusercontent.com/beniamincostas/Skill/main/commands/deep-review.md \
  -o ~/.claude/commands/deep-review.md
```

**Per-repo (shared with your team, versioned with the code):** copy
`commands/deep-review.md` into your repo's `.claude/commands/` and commit it.

Either way it becomes available as `/deep-review`.

## Usage

```
/deep-review                                  # auto-detect scope (diff, else whole codebase)
/deep-review review the whole product         # whole-product audit, grouped by subsystem
/deep-review src/auth                          # scope to a path / subsystem
/deep-review 1234                              # review PR #1234
/deep-review main...HEAD --effort high         # a commit range at lighter effort
/deep-review --fix                             # review, then apply the safe fixes (batched, test-gated)
```

Flags (anywhere in the arguments): `--effort low|medium|high|xhigh` (default
`xhigh`), `--fix` (apply corrections after the review), `--ui` / `--no-ui`
(force or skip the UI/a11y angle).

## What it does

- **Phase 0 — scope resolver.** PR / branch / range / path / subsystem /
  whole-product / not-a-git-repo, with an empty-scope guard, generated/binary
  exclusion, and a scale guard for large monorepos. The output cap scales with
  scope instead of being a fixed number.
- **Phase 1 — multi-angle fan-out.** Independent finder sub-agents:
  correctness (A–E), a connections/contracts/seams angle (the highest-value one
  for whole-product reviews), a UI/rendering/state/a11y angle (when a frontend
  exists), and cleanup / altitude / conventions angles.
- **Phase 2 — verify.** One verifier per candidate with a mandatory
  intent/by-design check, reachability check, live-vs-latent grounding, and a
  fix-complexity / regression-risk estimate.
- **Phase 3 — gap sweep** by a fresh reviewer.
- **Phase 4 — opt-in correction.** Capture a green baseline, fix in
  severity-ordered batches, re-run checks after each, revert on regression,
  defer the risky, re-verify observable changes (browser for UI).

## Notes

- It only runs when you type `/deep-review` (`disable-model-invocation: true`)
  — it's heavyweight and fans out sub-agents, so it never auto-triggers.
- It relies only on stock Claude Code (sub-agent fan-out, Bash, Read, Edit) —
  nothing to install.

> This command was distilled from a real whole-product review: a fixed
> diff-only assumption, a fixed findings cap, and a single-vote verify that
> confirmed intentional behavior all surfaced as concrete failure modes — which
> shaped the scope resolver, the scaling cap, and the intent/by-design check.
