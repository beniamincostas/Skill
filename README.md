# deep-review

A **scope-adaptive code-review (and optional auto-fix)** plugin for
[Claude Code](https://claude.com/claude-code). General-purpose — no repo,
language, framework, or topic baked in.

Point it at a PR, a branch, a commit range, a path, a named subsystem, or your
whole product (it even works in a non-git tree). It fans out independent finder
sub-agents, verifies every candidate before reporting it (intent + reachability
+ live-vs-latent grounding), sweeps for gaps, and emits ranked findings with
rich metadata. With `--fix` it then applies the safe corrections in
severity-ordered, test-gated batches and defers the risky ones — never shipping
a regression.

## Install (one time)

In Claude Code, run:

```
/plugin marketplace add beniamincostas/Skill
/plugin install deep-review@beniamincostas
```

That's it. Then use it anywhere:

```
/deep-review:deep-review
```

> The command is namespaced `plugin:command`, so it's `/deep-review:deep-review`
> (type `/deep-review` and pick it from the menu). Updates: re-run
> `/plugin marketplace update beniamincostas` and reinstall.

## Usage

```
/deep-review:deep-review                              # auto-detect scope (diff, else whole codebase)
/deep-review:deep-review review the whole product     # whole-product audit, grouped by subsystem
/deep-review:deep-review src/auth                     # scope to a path / subsystem
/deep-review:deep-review 1234                         # review PR #1234
/deep-review:deep-review main...HEAD --effort high    # a commit range at lighter effort
/deep-review:deep-review --fix                        # review, then apply the safe fixes (batched, test-gated)
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

## Manual install (no plugin)

Prefer to just drop in the command file? Copy it into your personal commands
(`~/.claude/commands/`, all projects) or a repo's `.claude/commands/`
(shared with your team):

```bash
mkdir -p ~/.claude/commands
curl -fsSL https://raw.githubusercontent.com/beniamincostas/Skill/main/plugins/deep-review/commands/deep-review.md \
  -o ~/.claude/commands/deep-review.md
```

Then it's available as `/deep-review` (un-namespaced).

## Notes

- It only runs when you invoke it (`disable-model-invocation: true`) — it's
  heavyweight and fans out sub-agents, so it never auto-triggers.
- It relies only on stock Claude Code (sub-agent fan-out, Bash, Read, Edit) —
  nothing else to install.

## Repo layout

```
.claude-plugin/marketplace.json          # marketplace (lists the deep-review plugin)
plugins/deep-review/
  .claude-plugin/plugin.json             # plugin manifest
  commands/deep-review.md                # the command
README.md
```

> Distilled from a real whole-product review: a fixed diff-only assumption, a
> fixed findings cap, and a single-vote verify that confirmed intentional
> behavior all surfaced as concrete failure modes — which shaped the scope
> resolver, the scaling cap, and the intent/by-design check.
