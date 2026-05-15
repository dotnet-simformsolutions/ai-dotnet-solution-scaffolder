---
name: tech-debt-remediation
description: 'Provide per-finding remediation solution templates, business impact phrasing, and solution rationale for tech debt findings. Load this skill when populating solution, solution_rationale, remediation, and business_impact fields for any DebtFinding. Triggers: solution template, remediation guidance, fix steps, business impact, why this approach, effort hours adjustment.'
user-invocable: false
---

# Tech Debt Remediation Templates

Use these templates as a **starting point** — always adapt them to reference the specific files,
versions, and infrastructure found in the codebase being analysed.

## Code Quality (CQ) Solutions

### TODO/FIXME/HACK/XXX Deferred Comments

**Solution**:
1. Run `grep -r "TODO\|FIXME\|HACK\|XXX" --include="*.{ext}" .` to produce a full list.
2. Create a backlog item in the project tracker for each occurrence, assigning to the last author
   via `git log -1 --format="%ae" {file}`.
3. Add a CI lint rule (e.g., `no-warning-comments` for JS, a custom Roslyn analyser for C#) that
   blocks new `TODO` additions without a linked ticket.

**Rationale**: Assigning via git blame ensures no deferred work is orphaned. CI enforcement
prevents regression without requiring manual audits.

**Business Impact**: "Signals accumulated scope creep — increases risk of unplanned production
defects and slows new feature delivery."

### Large Files (>500 lines)

**Solution**:
1. Identify cohesive sections in `{filename}` (groupings of methods that share a single purpose).
2. Extract each cohesive section into a new file following the existing naming convention.
3. Update all import/using references.
4. Run existing tests to confirm no breakage.
5. Aim for files under 300 lines.

**Rationale**: Decomposing by cohesion rather than arbitrary line count ensures each new file has
a single responsibility, making it independently testable and reviewable.

**Business Impact**: "Slows onboarding of new engineers and increases the risk of regression bugs
during maintenance."

---

### Magic Numbers (hardcoded literals in business logic)

**Solution**:
1. For each literal found, determine its semantic meaning from context.
2. Replace with a named constant at the top of the file or in a shared `Constants`/`config` file.
3. If the value is environment-specific, move it to configuration (appsettings / env variable).

**Rationale**: Named constants make intent explicit and eliminate duplication — changing one
business value updates every usage automatically.

**Business Impact**: "Increases maintenance cost: unnamed literals are a silent change-risk and
make code reviews harder."

### Deep Nesting (4+ levels of indentation)

**Solution**:
1. Apply early-return / guard-clause pattern at the top of each function (validate inputs first,
   return or throw immediately on failure).
2. Extract each deeply nested block into a private helper function with a descriptive name.
3. Use language-idiomatic collection pipelines (LINQ, streams) to replace manual loop nesting
   where applicable.

**Rationale**: Guard clauses reduce cognitive load by flattening the happy path; named helpers
document intent inline without requiring a separate file.

**Business Impact**: "Slows code reviews and increases defect rate — nested logic is hard to
test in isolation and error-prone to modify."

### Large Functions (>50 lines)

**Solution**:
1. Identify each logical sub-task within `{function}`.
2. Extract each sub-task into a private method named after the action it performs.
3. Keep the original function as an orchestrator that calls the helpers in sequence.
4. Target ≤ 30 lines per function.

**Rationale**: Orchestrator + helper decomposition is the lowest-risk refactor pattern because
the public API surface is unchanged and each helper can be unit-tested independently.

**Business Impact**: "Increases defect density in critical business logic — large functions
are difficult to test, review, and safely change."

## Dependency Health (DH) Solutions

### EOL Runtime

**Solution**:
1. Review the official migration guide for `{runtime}` from `{version}` to the current LTS.
2. Update the target framework moniker in all project files.
3. Run `{package_manager} build` to surface breaking API changes.
4. Fix compilation errors.
5. Run all tests.
6. Update the CI pipeline image/container to the new runtime version.

**Rationale**: Runtime upgrades must go through the full test suite because the runtime ships
behavioural changes. Updating CI at the same time prevents environment drift.

**Business Impact**: "Creates regulatory and security exposure — vendor support for CVE patches
ends at EOL, making the application an uninsurable risk in enterprise contracts."

### Outdated Packages (major version)

**Solution**:
1. Read the `CHANGELOG.md` / migration guide for `{package}` to identify breaking changes
   between `{current_version}` and `{latest_version}`.
2. Update the version reference.
3. Resolve compilation errors caused by renamed or removed APIs.
4. Run tests.

**Rationale**: Major bumps require manual migration because they intentionally break backwards
compatibility; reading the changelog before upgrading prevents trial-and-error debugging.

**Business Impact**: "Exposes the application to unpatched CVEs and increases long-term upgrade
cost as each skipped version compounds the migration effort."

### Outdated Packages (patch/minor)

**Solution**:
1. Run `{package_manager} update` for patch/minor outdated packages.
2. Run tests.
3. Add `{package_manager} audit` / `dotnet list package --vulnerable` to CI as a non-blocking
   warning step (promote to blocking once the team baseline is clean).

**Rationale**: Patch/minor upgrades are low-risk and can be batched; wiring the audit command
into CI makes vulnerability discovery continuous rather than point-in-time.

**Business Impact**: "Accumulates security exposure quietly — unpatched minor versions often
contain CVE fixes that are trivial to apply."

### Audit Vulnerabilities

**Solution**:
1. Run `{package_manager} audit` to enumerate all CVEs.
2. For auto-fixable items (`npm audit fix` or `pip install --upgrade`), apply and commit.
3. For items requiring major version bumps, open individual backlog items per package.
4. Add `--audit` to the CI dotnet/node/python step with a `--severity high` failure threshold.

**Rationale**: Separating auto-fixable from manual CVEs focuses developer effort on items that
require review, while automation handles the rest.

**Business Impact**: "Exposes the application and client data to known exploits — a liability
in enterprise procurement and compliance reviews."

## Test Coverage (TC) Solutions

### No Tests

**Solution**:
1. Add the recommended test framework (`jest` / `pytest` / `xunit`) to the project.
2. Create a `tests/` directory mirroring the source structure.
3. Write tests for the three most critical entry points (API routes / service methods /
   controllers).
4. Wire the test runner into CI with a coverage reporter.
5. Set an initial 40% coverage gate.

**Rationale**: Starting with the three most critical paths delivers the highest risk-reduction
per hour invested; the 40% initial gate is achievable without blocking delivery.

**Business Impact**: "Increases cost and risk of every future change; bugs reach production more
frequently, increasing breach and downtime risk for the client."

### Low Coverage (< 40%)

**Solution**:
1. Run the existing test suite with coverage enabled to identify uncovered files.
2. Prioritise test authoring for `{top_files}` (highest business value / highest change frequency).
3. Add a coverage gate to CI at the **current** percentage to prevent regression.
4. Incrementally raise the gate by 5% per sprint until reaching 60%.

**Rationale**: Setting the gate at the current level (rather than an aspirational target) is
immediately achievable, prevents regression, and creates visible progress.

**Business Impact**: "Low test coverage means every feature addition or bug fix carries hidden
regression risk that compounds over time."

### No Coverage Report

**Solution**:
1. Add `--collect:"XPlat Code Coverage"` to the `dotnet test` / `pytest --cov` / `jest
   --coverage` CI step.
2. Add a coverage-report publish step (e.g., `reportgenerator` for .NET, `coveralls` for
   Node/Python).
3. Display coverage trend on the CI dashboard.

**Rationale**: Coverage is only actionable when it is visible; publishing to the CI dashboard
gives the team a feedback loop without requiring manual runs.

**Business Impact**: "Without measurable coverage, the team cannot track quality improvement
or justify testing investment to stakeholders."

## Security Posture (SP) Solutions

### Hardcoded Credentials

**Solution**:
1. Immediately rotate **all** credentials in `{file_paths}` — treat as compromised regardless of
   repo visibility.
2. Add `{file}` to `.gitignore` and remove the tracked version with `git rm --cached {file}`.
3. Wire the secrets manager (e.g., Azure Key Vault at `{keyvault_url}`) to load secrets at
   runtime using the platform SDK (`Azure.Extensions.AspNetCore.Configuration.Secrets` / AWS
   Secrets Manager SDK / equivalent).
4. Create `{appsettings.template.json}` with placeholder values for developer onboarding.
5. Install `git-secrets` or `trufflehog` as a pre-commit hook to prevent future commits.
6. Run `git filter-repo --path {file} --invert-paths` to scrub history.

**Rationale**: Key Vault / Secrets Manager provides audit logging, rotation workflows, and
access control at no extra cost — wiring it replaces credentials with references, eliminating
the root cause rather than just removing the file.

**Business Impact**: "A critical security incident risk — exposed credentials can lead to data
breach, regulatory fines, and lasting reputational damage for the client."

### Dangerous Execution Functions (eval/exec/shell=True)

**Solution**:
1. For each `eval`/`exec`/`os.system` call in `{file}`, determine whether the input is fully
   controlled (literal string) or user-influenced.
2. If user-influenced: replace with a safe alternative (parameterized subprocess, AST parser,
   explicit allow-list).
3. If unavoidable: add input validation against an explicit allow-list and wrap in a try/catch
   with logging.
4. Add a CodeQL or Semgrep rule to block new `eval` additions in CI.

**Rationale**: Allow-list validation is safer than deny-list because new attack vectors
automatically fall outside the allowed set; CI enforcement prevents regression.

**Business Impact**: "Remote code execution risk — user-controlled input reaching these
functions is a critical vulnerability that can compromise the entire server."

### Plain HTTP URLs (non-TLS)

**Solution**:
1. Replace all `http://` URLs in `{file}` with `https://`.
2. Add an HSTS header at the application or infrastructure layer.
3. Add a grep-based CI check that blocks new `http://` additions (excluding localhost / test URLs).

**Rationale**: HTTPS is mandatory for any data in transit; HSTS prevents protocol downgrade
attacks even if a URL is accidentally hardcoded without the scheme upgrade.

**Business Impact**: "Transmits data in cleartext — a compliance failure under GDPR, PCI-DSS,
and HIPAA regimes, and trivially exploitable on shared networks."

### Committed .env Files

**Solution**:
1. Add `*.env` and `.env.*` to `.gitignore` immediately.
2. Run `git rm --cached {file}` to stop tracking the file.
3. Rotate all credentials referenced in the file.
4. Purge the file from history with `git filter-repo --path {file} --invert-paths`.
5. Add a pre-commit hook (`detect-secrets` or `git-secrets`) to block future `.env` commits.

**Rationale**: Removing from `.gitignore` without purging history leaves credentials accessible
via `git log`; `git filter-repo` is the modern, supported replacement for `git filter-branch`.

**Business Impact**: "Credentials in version history are permanently accessible to anyone with
repo access — a compliance and incident-response liability."

### SQL Injection Patterns

**Solution**:
1. Replace every string-concatenated query in `{file}` with a parameterized query using the
   existing ORM / database driver.
2. For raw SQL that cannot be parameterized, use explicit allow-list validation on user input.
3. Add a Semgrep / CodeQL rule for SQL-by-string-concat to CI.
4. Run OWASP ZAP or SQLMap against a staging environment to verify no injection surfaces remain.

**Rationale**: Parameterized queries are the only fully reliable mitigation — allow-list
validation is a secondary defence. Using the existing ORM means no new dependencies.

**Business Impact**: "Exposes all client data to exfiltration or corruption — a critical
compliance and liability risk that can void cyber-insurance coverage."

## Architecture (AR) Solutions

### No Layer Separation

**Solution**:
1. Define the target folder structure (e.g., `controllers/`, `services/`, `repositories/`,
   `models/`).
2. Move **new** code into the correct layer immediately (zero-cost enforcement from day one).
3. Incrementally migrate the three highest-traffic modules per sprint.
4. Add an import-boundary lint rule (e.g., `eslint-plugin-import`, NDepend) to enforce layer
   boundaries in CI.

**Rationale**: Incremental migration is safer than a big-bang refactor because the application
remains shippable throughout; enforcing boundaries in CI prevents the structure from reverting
under delivery pressure.

**Business Impact**: "Increases the cost of every new feature — tightly coupled systems are
expensive to scale, test, and extend as the client's product grows."

### Monolithic Files (>1000 lines, multiple classes)

**Solution**:
1. List all classes/structs defined in `{file}`.
2. Move each non-primary class to its own file following the existing naming convention.
3. Update all references.
4. Run existing tests to confirm no breakage.

**Rationale**: One-class-per-file is the lowest-risk decomposition step and is a prerequisite
for meaningful per-class test coverage.

**Business Impact**: "Slows every code review and merge, and makes parallel development on the
same area impossible without high merge-conflict risk."

### Cross-Layer Imports (circular dependency indicators)

**Solution**:
1. Generate a dependency graph (`madge` for JS/TS, `pydeps` for Python, NDepend for C#).
2. Identify cycles.
3. Break each cycle by extracting the shared logic into a new, dependency-free utility module
   that both layers can import.

**Rationale**: Extracting shared logic rather than changing import direction is the safest
resolution because it does not require existing callers to change their call sites.

**Business Impact**: "Circular dependencies make the codebase fragile — a change in one layer
can cause cascading failures in unrelated areas."
