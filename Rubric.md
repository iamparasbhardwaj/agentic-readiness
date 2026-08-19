# Agent Readiness Rubric v0.1

Scope: web applications and REST APIs. Not calibrated for libraries, CLIs, mobile
apps, or data pipelines — those will fail checks that don't apply to them.

Total scored points: **37** across 20 checks, plus 2 unscored gates.

Every check has three possible results:

- **PASS** — evidence found
- **FAIL** — looked, found nothing
- **UNVERIFIED** — couldn't look (no `gh`, not authenticated, not a GitHub remote,
  permission denied, API error)

UNVERIFIED is removed from both the numerator and the denominator. Never guess a
pass or a fail from absence of access — the whole point of the third state is that
a blind spot is visible instead of silently distorting the score.

---

## TIER 0 — Gates (no points)

These don't add score. They cap it. A repo that fails either gate cannot be rated
above **Supervised**, however well it scores elsewhere, because the failure mode
they prevent is an agent pushing straight to production.

If a gate is UNVERIFIED, do not apply the cap — report the gate as unknown and note
that the band is provisional.

### G1 — protected_main

Default branch requires a pull request and at least one approving review.

- `gh api repos/{owner}/{repo}/branches/{branch}/protection` — look for
  `required_pull_request_reviews.required_approving_review_count >= 1`
- A 404 here usually means no protection is configured → FAIL
- A 403 means insufficient token scope → UNVERIFIED
- Also check for a `.github/CODEOWNERS`-backed required review if present

### G2 — secret_hygiene

Secrets aren't in the repo, and something is watching for them.

- `.gitignore` excludes `.env`, `.env.local`, `*.pem`, `*.key` (but permits
  `.env.example`)
- Any of: `gh api repos/{owner}/{repo}` → `security_and_analysis.secret_scanning.status`
  is `enabled`; a `gitleaks`/`trufflehog`/`detect-secrets` config; a secret-scanning
  step in CI; a `git-secrets` pre-commit hook
- Spot-check tracked files for hardcoded credentials —
  `git grep -nE '(api[_-]?key|secret|password|token)\s*[:=]\s*["'"'"'][^"'"'"']{12,}'`
  and ignore test fixtures, examples, and placeholder values
- **When reporting a suspected hardcoded secret, give the file and line only.
  Never reproduce the value in the report.**

---

## TIER 1 — Foundations (3 points each, 15 total)

Any FAIL in this tier caps the repo at **Delegate**. These are the checks that
decide whether an agent can get the project running and verify its own work at all.
Everything in Tier 2 and 3 assumes these hold.

### T1.1 — one_command_setup (3)

A single documented command takes a fresh clone to a running app.

- `README.md` or `AGENTS.md` contains a runnable setup line: `docker compose up`,
  `make setup`, `npm ci && npm run dev`, `uv sync && uv run …`
- Or a `Makefile` with a `setup`/`bootstrap`/`dev` target
- Or `package.json` scripts containing `dev`/`start` plus documented prerequisites
- A README that lists eight sequential manual steps is a FAIL — the criterion is
  *one* command, because each extra step is a place the agent guesses wrong

### T1.2 — one_command_test (3)

A single documented command runs the full test suite.

- `npm test` / `pnpm test` in `package.json` scripts
- `pytest` configured in `pyproject.toml`, `pytest.ini`, or `setup.cfg`
- A `test` target in `Makefile`, `justfile`, or `Taskfile.yml`
- The command should be named in the README or AGENTS.md, not only inferable

### T1.3 — env_template (3)

A committed sample environment file lists every setting the app reads, by name.

- `.env.example`, `.env.sample`, `.env.template`, or an explicit environment
  variables section in README/AGENTS.md
- Sanity check: grep the source for `process.env.` / `os.environ` / `os.getenv` and
  compare the variable names found against the template. If the code reads names
  the template doesn't mention, FAIL — a partial template is the failure mode this
  check exists to catch, because the agent trusts it and then guesses the rest
- The template must hold placeholders, not real values

### T1.4 — agents_md (3)

An agent instruction file exists and says something useful.

- Any of `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `.github/copilot-instructions.md`
- Must cover at least two of: the commands to run, code conventions, project
  layout, gotchas
- A file that only says "this is a Node project" is a FAIL. Judgment call — record
  which of the four it covers so the call is auditable

### T1.5 — readme_prescriptive (3)

`README.md` covers the stack, how to run it, and how to test it — and is current.

- File exists and is longer than roughly 30 lines of substance
- Mentions the stack, has a setup or install section, has a run or test command
- `git log -1 --format=%ci -- README.md` is within 180 days
- A README last touched two years ago describing a framework version you no longer
  use is worse than no README, since the agent will believe it

---

## TIER 2 — Feedback loop (2 points each, 14 total)

This tier is the difference between an agent that writes code and one that fixes
its own code. Each item shortens the gap between a mistake and the agent finding out.

### T2.1 — linter_and_formatter (2)

Both configured at the root.

- JS/TS: `eslint.config.*`, `.eslintrc*`, `biome.json`, `.prettierrc*`, or a
  `prettier`/`eslint` key in `package.json`
- Python: `[tool.ruff]`, `[tool.black]`, `.flake8`, `setup.cfg` sections
- Go: `.golangci.yml`
- Configured, not just present in dependencies

### T2.2 — type_checking_enforced (2)

Type checking runs as a gate, not as decoration.

- `tsconfig.json` with `"strict": true`, or mypy/pyright with a strict setting
- **And** invoked somewhere that blocks: a CI step, a pre-commit hook, or a
  `typecheck` script wired into `test`
- `strict: false`, or a config with no runner, is a FAIL — installed-but-unenforced
  is the common case and it gives the agent no signal

### T2.3 — local_precommit_gate (2)

Checks run on the developer's machine before the commit lands.

- `.pre-commit-config.yaml`
- Or `.husky/` with a `pre-commit` hook
- Or `lint-staged` config
- Or `core.hooksPath` set with hooks committed to the repo
- CI-only linting means the agent waits minutes to learn about a missing semicolon,
  then burns a round trip fixing it blind

### T2.4 — selective_test_execution (2)

You can run one test or one directory rather than the whole suite.

- Test runner supports it natively (pytest, jest, vitest, go test all do) **and**
  nothing in the config forces a full run
- Look for a documented example of running a subset in README/AGENTS.md
- FAIL if the only entry point is a monolithic script that always runs everything,
  or if tests depend on global setup that can't be scoped

### T2.5 — fast_test_feedback (2)

The suite finishes fast enough to sit inside an iteration loop.

- `gh run list --limit 20 --json durationMs,conclusion,workflowName` — take the
  median duration of successful test workflows
- PASS under 10 minutes, FAIL over 10
- If `gh` is unavailable, fall back to a timeout setting in the CI config; if
  there's nothing to read, mark UNVERIFIED rather than assuming

### T2.6 — isolated_test_state (2)

Tests don't touch live external services or shared data.

- A mocking library configured: `msw`, `nock`, `responses`, `vcr`, `wiremock`,
  `httpretty`
- Or a test database: a `docker-compose.test.yml`, a test service in CI,
  `testcontainers`, an SQLite/in-memory test config
- Or fixtures plus a documented test database setup
- FAIL if test config points at a real hostname or a shared staging database

### T2.7 — ci_on_every_pr (2)

CI runs automatically on pull request commits.

- `.github/workflows/*.yml` with `on: pull_request` (or `.gitlab-ci.yml`,
  `.circleci/config.yml` equivalent)
- The workflow must actually run tests or checks, not only build a container or
  post a label

---

## TIER 3 — Amplifiers (1 point each, 8 total)

Real value, but never a substitute for the tiers above. A repo scoring high here
and failing Tier 1 is a repo with good paperwork and no working feedback loop.

### T3.1 — deps_locked (1)

A committed lockfile: `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`,
`uv.lock`, `poetry.lock`, `Pipfile.lock`, `go.sum`, `Gemfile.lock`.

### T3.2 — api_contract (1)

A machine-readable API description: an `openapi.{json,yaml}` or `swagger.*` file, a
framework that generates one (FastAPI, NestJS Swagger, drf-spectacular), or
committed `.proto` / GraphQL schema files.

### T3.3 — versioned_migrations (1)

A migrations directory with ordered files (Alembic, Prisma, Django, Flyway,
Knex, ActiveRecord) **and** a documented command to apply them.

### T3.4 — coverage_gate (1)

A coverage minimum that fails the build — `--fail-under`, `coverageThreshold`,
`--cov-fail-under`, a Codecov `target` with `informational: false`. Measuring
coverage without a threshold doesn't count.

### T3.5 — runtime_visibility (1)

Something is configured to explain a runtime failure: a structured logging library
(`pino`, `winston`, `structlog`, `zap`, `zerolog`) or an error tracker (Sentry,
Rollbar, Bugsnag, Honeybadger) with initialization in the source.

### T3.6 — codeowners (1)

`CODEOWNERS` in the root, `.github/`, or `docs/`, with at least one non-comment
ownership line.

### T3.7 — reproducible_env (1)

`.devcontainer/devcontainer.json`, a development `Dockerfile` plus
`docker-compose.yml`, a `flake.nix`, or `mise.toml` / `.tool-versions` pinning
runtime versions.

### T3.8 — issue_and_pr_templates (1)

`.github/ISSUE_TEMPLATE/` (or `ISSUE_TEMPLATE.md`) **and** a
`pull_request_template.md`. Both needed for the point — this is a proxy for
whether incoming work arrives structured, which is the closest a repo scan can get
to ticket quality.

---

## Scoring

```
earned    = sum of points from PASS checks
available = 37 − (points of UNVERIFIED checks)
percentage = earned / available
```

Then apply caps in order. A cap lowers a band; it never raises one.

- Any Tier 0 gate FAIL → cap at **Supervised**
- Any Tier 1 check FAIL → cap at **Delegate**

### Bands

| Band | Range | What it means |
|---|---|---|
| **Not ready** | < 40% | The agent will guess how the project works, produce large confident diffs, and spend budget without producing anything mergeable. |
| **Supervised** | 40–64% | Small isolated changes only. You run the tests yourself and clean up formatting by hand. |
| **Delegate** | 65–84% | Hand it a real multi-file ticket. It can verify and mostly correct its own work. You still read the PR properly. |
| **High autonomy** | 85%+, no Tier 1 FAIL | Multi-file work in a sandbox, reviewed as a finished PR. |

Always report the band **with** the raw fraction — "Delegate, 24/37" — and list any
cap that fired. A band alone invites the argument where someone claims a level the
evidence doesn't support.

---

## Changelog

- **v0.1** — initial. Web app / REST API only. 20 scored checks, 2 gates, 37 points.
