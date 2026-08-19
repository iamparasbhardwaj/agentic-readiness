---
name: agent-readiness
description: Grade a local repository on how well it supports autonomous coding agents, scoring it against a 37-point rubric across setup, testing, feedback loops, and safety guardrails, and produce a report with a band, evidence for every check, and a prioritized fix list. Use this skill whenever the user asks how agent-ready, AI-ready, or agent-friendly a repo is, asks for a readiness report or readiness score, asks why an agent is performing badly in their codebase, asks what to fix before letting an agent work autonomously, or asks to audit a repo for AI coding agents — even if they don't use the word "readiness". Scoped to web applications and REST APIs.
---

# Agent Readiness

Grade a repository on whether an autonomous coding agent can actually work in it.

The premise: most agent underperformance is an environment problem, not a model
problem. An agent that can't figure out how to boot the app will guess, fail, and
guess again. An agent with no fast local check has to wait on CI to learn it wrote a
syntax error. This skill measures those conditions and says what to fix first.

## Before starting

Read `rubric.md` in this skill directory. It holds all 22 checks, their detection
hints, point values, and the banding rules. It is the source of truth for scoring —
don't score from memory, and don't invent checks that aren't in it.

Confirm the target is in scope. The rubric is calibrated for **web applications and
REST APIs**. If the repo is a library, CLI tool, mobile app, or data pipeline, say so
before grading and offer either to grade anyway with the caveat that several checks
structurally cannot pass, or to stop. Grading an out-of-scope repo and reporting a
low band as if it were meaningful is the main way a rubric like this loses trust.

## Setting up

Work from the repo root. Establish these once, up front, since most checks depend on
them:

```bash
git rev-parse --show-toplevel          # confirm we're in a repo
git remote get-url origin              # derive owner/repo for gh calls
git branch --show-current              # and the default branch
gh auth status                         # is gh available and authenticated?
```

`gh` is optional. Six checks read from the GitHub API (both Tier 0 gates, plus
`fast_test_feedback`) and the rest are filesystem-only. If `gh` is missing, not
authenticated, or the remote isn't GitHub, mark those checks **UNVERIFIED** and carry
on. Do not install anything, and do not treat a missing tool as a failing repo.

Then get oriented before checking anything:

```bash
ls -a                                  # root config files
cat package.json pyproject.toml go.mod 2>/dev/null | head -60
ls .github/workflows/ 2>/dev/null
```

Knowing the stack first stops you from marking a Python project as missing ESLint.

## Running the checks

Work tier by tier — gates, then Tier 1, then 2, then 3. Order matters for reporting,
and Tier 1 failures change the band regardless of what Tier 3 says.

Prefer cheap deterministic evidence. Most checks resolve with `ls`, `cat`, a
targeted `grep`, or one `gh api` call. Reach for judgment only where the rubric
explicitly calls for it (`agents_md`, `readme_prescriptive`, `one_command_setup`),
and when you do, record what you saw so the call can be audited later.

Batch reads where you can — one `ls -a` and one combined `cat` beats fifteen
separate existence checks.

**Every result needs evidence.** For a PASS, name the file and the specific thing
found in it: not "linter configured" but "Ruff configured in pyproject.toml
[tool.ruff], select = E,F,I". For a FAIL, name what you looked for and didn't find.
For UNVERIFIED, name what blocked you. Someone will disagree with their score, and
the evidence line is what ends that conversation in ten seconds instead of ten
minutes.

Two things to be careful about:

- **Configured vs. enforced.** A `tsconfig.json` with strict mode that nothing runs
  is a FAIL. A coverage tool with no threshold is a FAIL. The rubric flags these
  individually; the general principle is that a check the agent never sees the output
  of provides the agent nothing.
- **Suspected secrets.** If the secret-hygiene grep surfaces something, report the
  file and line number only. Never put the value in the report.

## Scoring

Follow `rubric.md` exactly:

1. Sum points from PASS checks.
2. Subtract UNVERIFIED check points from the 37-point denominator.
3. Percentage of what was available, not of 37.
4. Apply caps: any gate FAIL caps at Supervised; any Tier 1 FAIL caps at Delegate.
   Caps only lower a band.
5. If a gate is UNVERIFIED, don't apply its cap — mark the band provisional and say
   what access would resolve it.

Report the band together with the raw fraction, always.

## Report structure

Use this template:

```markdown
# Agent Readiness — [repo name]

**[Band]** — [earned]/[available] points ([percentage]%)
[Any cap that fired, and why. Or: "No caps applied."]
[If anything was UNVERIFIED: what, and what access would resolve it.]

## Fix these first

1. **[Check name]** — [what's missing, one line] → [+N points]
2. ...
3. ...

## Gates

| Check | Result | Evidence |
|---|---|---|

## Tier 1 — Foundations (n/15)

| Check | Result | Evidence |
|---|---|---|

## Tier 2 — Feedback loop (n/14)
[same shape]

## Tier 3 — Amplifiers (n/8)
[same shape]
```

Order the fix list by points-per-effort, not by point value. Writing an
`.env.example` is worth 3 points and takes twenty minutes; getting a flaky suite
deterministic is worth 2 and takes a month. Lead with the cheap wins, and call out
separately any single failure that's capping the band — that one is usually worth
doing regardless of cost, because it's holding down everything else.

Keep the fix list to three items. A twenty-item list gets read once and ignored.

## Deliberately out of scope for v0.1

Don't extend into these without the user asking:

- Fixing anything, or opening a PR. This skill grades; it doesn't remediate.
- Monorepos. If the repo has multiple apps, grade the whole thing once and say the
  score is coarse.
- Comparing against a previous run. There's no history to ground on yet.
- Non-web-app project types.

## When the rubric is wrong

It will be, at first. If a check keeps producing a result that doesn't match your
read of a repo — a project you know is well-run scoring badly, or a mess scoring
well — that's a rubric problem, not a scoring problem. Say so in the report and
suggest the specific wording change. Editing `rubric.md` is cheap and expected;
quietly compensating for a bad check by fudging the result is not, because it makes
the score unreproducible.
