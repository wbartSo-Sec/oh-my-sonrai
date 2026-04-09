# [SAR-ID] [Story Title]

**Epic**: [Epic Jira Link]  
**Repository**: [repo-name]  
**Story Link**: [Story Jira Link]

---

## TL;DR

> **Quick Summary**: [1-2 sentences capturing the core objective and approach for this Story]
>
> **Deliverables**: [Bullet list of concrete outputs]
> - [Output 1]
> - [Output 2]
> - [Output 3]
>
> **Estimated Effort**: [Quick | Short | Medium | Large | XL]  
> **Parallel Execution**: [YES - N waves | NO - sequential]  
> **Critical Path**: [Task X → Task Y → Task Z]

---

## Context

### Epic Context
[Context about the overarching Epic this Story belongs to. What problem does the Epic solve? Why does it matter?]

### This Story's Role
[How does this specific Story fit into the Epic? What dependencies does it have on other Stories?]

### Affected Repositories
- [repo-1]: [Change summary]
- [repo-2]: [Change summary]

---

## Feature Flag Strategy

**Flag Name Pattern**: `ff-[SAR-ID]-[feature]`  
**Example**: `ff-SAR-12345-new-auth-flow`

**Default State**: disabled

**Rollout Stages**:
1. **dev**: Enable in dev environment (Flux PR #[number])
2. **stage**: Enable in stage environment (Flux PR #[number])
3. **limited-prod**: Enable for [X]% of production traffic (Flux PR #[number])
4. **prod**: Enable for 100% of production (Flux PR #[number])

**Kill Switch Instructions**:
- To disable immediately: Run `helm upgrade [release] [chart] --set featureFlags.ff-[SAR-ID]-[feature]=false`
- To verify state: `kubectl get configmap [config-name] -o yaml | grep ff-[SAR-ID]`
- Flux PR to revert: [Flux PR Link]

---

## Work Objectives

### Core Objective
[1-2 sentences: What are we achieving? What problem does this Story solve?]

### Concrete Deliverables
- [Exact file path or endpoint created]
- [Exact behavior implemented]
- [Exact integration points]

### Definition of Done
- [ ] [Verifiable condition with exact command/assertion]
- [ ] [Another verifiable condition]
- [ ] [Third verifiable condition]

---

## Must Have / Must NOT Have

### Must Have
- [Non-negotiable requirement 1]
- [Non-negotiable requirement 2]
- [Non-negotiable requirement 3]

### Must NOT Have (Guardrails)
- [Explicit exclusion — pattern to avoid from Metis review]
- [AI slop pattern — e.g., "no placeholder implementations", "no TODO comments in production code"]
- [Scope boundary — e.g., "no changes to authentication system", "database migration not in scope"]

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.  
> Acceptance criteria requiring "user manually tests/confirms" are FORBIDDEN.

### Test Decision
- **Infrastructure exists**: [YES | NO]
- **Automated tests**: [TDD / Tests-after / None]
- **Framework**: [bun test | vitest | jest | pytest | none]
- **If TDD**: Each task follows RED (failing test) → GREEN (minimal impl) → REFACTOR

### QA Policy

Every task MUST include agent-executed QA scenarios. Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **Frontend/UI**: Use Playwright (playwright skill) — Navigate, interact, assert DOM, screenshot
- **TUI/CLI**: Use interactive_bash (tmux) — Run command, send keystrokes, validate output
- **API/Backend**: Use Bash (curl) — Send requests, assert status + response fields
- **Library/Module**: Use Bash (bun/node REPL) — Import, call functions, compare output

---

## Execution Strategy

### Parallel Execution Waves

> Maximize throughput by grouping independent tasks into parallel waves.  
> Each wave completes before the next begins.  
> Target: 5-8 tasks per wave. Fewer than 3 per wave (except final) = under-splitting.

```
Wave 1 (Start Immediately — foundation + scaffolding):
├── Task 1: [Task title] [quick]
├── Task 2: [Task title] [quick]
├── Task 3: [Task title] [quick]
├── Task 4: [Task title] [quick]
└── Task 5: [Task title] [quick]

Wave 2 (After Wave 1 — core modules):
├── Task 6: [Task title] (depends: 1, 2) [deep]
├── Task 7: [Task title] (depends: 3) [unspecified-high]
├── Task 8: [Task title] (depends: 4, 5) [unspecified-high]
└── Task 9: [Task title] (depends: 6) [deep]

Wave FINAL (After ALL tasks — 4 parallel reviews, then approval):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Code quality review (unspecified-high)
├── Task F3: Real manual QA (unspecified-high)
└── Task F4: Scope fidelity check (deep)
-> Present results -> Get explicit user okay
```

### Dependency Matrix

- **1-5**: — — 6-9
- **6**: 1, 2 — 9
- **7**: 3 — 9
- **8**: 4, 5 — 9
- **9**: 6, 7, 8 — FINAL

---

## TODOs

> Implementation + Test = ONE Task. Never separate.  
> EVERY task MUST have: Recommended Agent Profile + Parallelization info + QA Scenarios.  
> **A task WITHOUT QA Scenarios is INCOMPLETE. No exceptions.**

- [ ] 1. [Task Title]

  **What to do**:
  - [Clear implementation step 1]
  - [Clear implementation step 2]
  - [Test case or behavior to implement]

  **Must NOT do**:
  - [Specific exclusion from guardrails]
  - [Pattern to avoid]

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `[visual-engineering | ultrabrain | artistry | quick | unspecified-low | unspecified-high | writing]`
    - Reason: [Why this category fits the task domain]
  - **Skills**: [`skill-1`, `skill-2`]
    - `skill-1`: [Why needed - domain overlap explanation]
    - `skill-2`: [Why needed - domain overlap explanation]
  - **Skills Evaluated but Omitted**:
    - `omitted-skill`: [Why domain doesn't overlap]

  **Parallelization**:
  - **Can Run In Parallel**: YES | NO
  - **Parallel Group**: Wave N (with Tasks X, Y) | Sequential
  - **Blocks**: [Tasks that depend on this task completing]
  - **Blocked By**: [Tasks this depends on] | None (can start immediately)

  **References** (CRITICAL - Be Exhaustive):

  > The executor has NO context from your interview. References are their ONLY guide.  
  > Each reference must answer: "What should I look at and WHY?"

  **Pattern References** (existing code to follow):
  - `[file-path]:[line-range]` - [What pattern to extract and why]

  **API/Type References** (contracts to implement against):
  - `[file-path]:[TypeName]` - [Response shape or contract]

  **Test References** (testing patterns to follow):
  - `[test-file-path]:[describe-block]` - [Test structure to follow]

  **External References** (libraries and frameworks):
  - [Official docs link] - [What to learn from this]

  **Acceptance Criteria**:

  > **AGENT-EXECUTABLE VERIFICATION ONLY** — No human action permitted.  
  > Every criterion MUST be verifiable by running a command or using a tool.

  **If TDD (tests enabled):**
  - [ ] Test file created: [path]
  - [ ] bun test [path] → PASS ([N] tests, 0 failures)

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  > **This is NOT optional. A task without QA scenarios WILL BE REJECTED.**
  >
  > Write scenario tests that verify the ACTUAL BEHAVIOR of what you built.  
  > Minimum: 1 happy path + 1 failure/edge case per task.  
  > Each scenario = exact tool + exact steps + exact assertions + evidence path.
  >
  > **The executing agent MUST run these scenarios after implementation.**  
  > **The orchestrator WILL verify evidence files exist before marking task complete.**

  ```
  Scenario: [Happy path — what SHOULD work]
    Tool: [Playwright | interactive_bash | Bash (curl)]
    Preconditions: [Exact setup state]
    Steps:
      1. [Exact action — specific command/selector/endpoint, no vagueness]
      2. [Next action — with expected intermediate state]
      3. [Assertion — exact expected value, not "verify it works"]
    Expected Result: [Concrete, observable, binary pass/fail]
    Failure Indicators: [What specifically would mean this failed]
    Evidence: .sisyphus/evidence/task-{N}-{scenario-slug}.{ext}

  Scenario: [Failure/edge case — what SHOULD fail gracefully]
    Tool: [same format]
    Preconditions: [Invalid input / missing dependency / error state]
    Steps:
      1. [Trigger the error condition]
      2. [Assert error is handled correctly]
    Expected Result: [Graceful failure with correct error message/code]
    Evidence: .sisyphus/evidence/task-{N}-{scenario-slug}-error.{ext}
  ```

  > **Specificity requirements — every scenario MUST use:**
  > - **Selectors**: Specific CSS selectors (`.login-button`, not "the login button")
  > - **Data**: Concrete test data (`"test@example.com"`, not `"[email]"`)
  > - **Assertions**: Exact values (`text contains "Welcome back"`, not "verify it works")
  > - **Timing**: Wait conditions where relevant (`timeout: 10s`)
  > - **Negative**: At least ONE failure/error scenario per task

  **Evidence to Capture:**
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES | NO (groups with [other task numbers if applicable])
  - Message: `type(scope): desc`
  - Files: `path/to/file`
  - Pre-commit: `test command`

---

## Final Verification Wave

> 4 review agents run in PARALLEL. ALL must APPROVE. Present consolidated results to user and get explicit "okay" before completing.
>
> **Do NOT auto-proceed after verification. Wait for user's explicit approval.**  
> **Never mark F1-F4 as checked before getting user's okay.** Rejection → fix → re-run → present → wait for okay.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (read file, curl endpoint, run command). For each "Must NOT Have": search codebase for forbidden patterns — reject with file:line if found. Check evidence files exist in .sisyphus/evidence/. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `tsc --noEmit` + linter + `bun test`. Review all changed files for: `as any`/`@ts-ignore`, empty catches, console.log in prod, commented-out code, unused imports. Check AI slop: excessive comments, over-abstraction, generic names (data/result/item/temp).
  Output: `Build [PASS/FAIL] | Lint [PASS/FAIL] | Tests [N pass/N fail] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high` (+ `playwright` skill if UI)
  Start from clean state. Execute EVERY QA scenario from EVERY task — follow exact steps, capture evidence. Test cross-task integration (features working together, not isolation). Test edge cases: empty state, invalid input, rapid actions. Save to `.sisyphus/evidence/final-qa/`.
  Output: `Scenarios [N/N pass] | Integration [N/N] | Edge Cases [N tested] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff (git log/diff). Verify 1:1 — everything in spec was built (no missing), nothing beyond spec was built (no creep). Check "Must NOT do" compliance. Detect cross-task contamination: Task N touching Task M's files. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Contamination [CLEAN/N issues] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Documentation & Handoff

### Documentation Generation
- [ ] Update [relevant docs file]: [description of update]
- [ ] Generate API docs: [command]
- [ ] Update README: [section reference]

### Change Order References
- **Stage CO**: [Change Order Ticket Link]
- **Prod CO**: [Change Order Ticket Link]

### Jira Workflow Transitions
After final verification wave approval, transition [SAR-ID] through:
1. `Discovery` → `Ready for Dev` (at task start)
2. `Ready for Dev` → `In Progress` (at Wave 1 start)
3. `In Progress` → `Development Complete` (when F1-F4 all approved)
4. `Development Complete` → `Stage Validation` (when deployed to stage)
5. `Stage Validation` → `Limited Prod` (when stage testing passes)
6. `Limited Prod` → `Prod Released` (when prod rollout completes)

---

## Commit Strategy

**Wave 1 Tasks**:
- Commit message: `type(scope): brief description`
- Pre-commit test: [test command]

**Wave 2 Tasks**:
- Commit message: `type(scope): brief description`
- Pre-commit test: [test command]

**Final Verification**:
- Commit message: `type(scope): final verification and integration`
- All tasks grouped into atomic commits per wave

---

## Dev Lead Approval Gate

**BEFORE** feature code begins (after Task 1 scaffolding only):

- [ ] **Draft PR Created**: Link to initial PR with Task 1 changes (scaffolding, config, types)
- [ ] **Dev Lead Review**: [Dev Lead Name] has reviewed and approved the architectural direction
- [ ] **Approval Checkpoint**: Dev Lead explicitly approves before Tasks 2-N begin

**Approval Checklist** (for Dev Lead):
- Architecture aligns with repo standards
- No scope creep detected
- Dependencies properly declared
- Feature flag strategy is sound
- Estimated effort is realistic

---
