# Unfog v2 implementation and proof plan

Status: proposed execution plan  
Owner: Unfog maintainers  
Target repository: `unfog-deploy`  
Primary artifact: `skill/`  

## 1. Outcome

Build Unfog v2 into an observable execution-control system that:

1. recovers the correct outcome, target, premise, scope, preservation boundary, acceptance probes, and delivery chain from underspecified work;
2. asks only when a material unresolved fork remains;
3. prevents risky work from mutating its target before its contract is ready;
4. prevents a completion claim when required probes lack valid receipts;
5. revises the contract explicitly when evidence or the user changes the task;
6. adds little or no process overhead to simple work; and
7. reports its performance from a fixed, reviewable population without hiding failures or changing denominators.

This plan does **not** promise the absence of every possible defect. That claim cannot be proven. “Works without errors” will mean:

- zero known unhandled failures in the disclosed release test matrix;
- zero invalid ready or complete contracts accepted by the adversarial fixture suite;
- zero unsafe scope expansions in the frozen transcript evaluation corpus;
- every observed failure and exclusion reported with its denominator; and
- the measured field-trial error rate remaining below the release threshold.

Unknown defects remain possible and must be described as residual risk rather than silently converted into success.

## 2. Baseline and evidence limits

The initial local audit inspected 162 Codex session files dated August 29 through September 7, 2026.

- 53 tasks contained direct assistant use of Unfog.
- Those tasks contained 191 explicit assistant references to Unfog.
- 4 tasks invoked the bundled persistent-contract helper.
- 1 task attempted `--phase complete` validation.
- The current validator passes 11 of 11 unit tests.

These figures establish an adoption and test baseline. They do not establish causality: the tasks were not randomly assigned, task difficulty was uncontrolled, and different models or environments may have affected outcomes. Wave 0 will make the extraction reproducible and create matched controls before any effectiveness claim is made.

## 3. Non-negotiable truth rules

Every implementation and evaluation decision must follow these rules.

### 3.1 Fixed populations

- Freeze each evaluation corpus before running a comparison.
- Give every case a stable ID and content digest.
- Report `eligible`, `attempted`, `passed`, `failed`, `blocked`, and `excluded` counts.
- Never remove a failed case from the denominator. A later correction creates a new run; it does not rewrite the old run.
- An exclusion requires a coded reason and remains visible in the report.
- A result over zero eligible or attempted cases is invalid.

### 3.2 No cherry-picking

- Select historical tasks by a documented sampling procedure, not by whether Unfog looked good.
- Stratify by work mode, risk, repository count, task length, and whether deployment or external state was involved.
- Keep a held-out test set that is not used while tuning prompts, schema rules, or thresholds.
- Report per-stratum results as well as the aggregate. Aggregate improvement cannot conceal a regression in destructive, security-sensitive, or production work.

### 3.3 Immutable raw evidence

- Store task IDs, source hashes, timestamps, commands, exit codes, and evaluator decisions.
- Never overwrite an evaluation run. A rerun gets a new run ID and references its predecessor.
- Separate raw evidence, derived metrics, and human judgments.
- Human labels require a rubric version and reviewer identity.
- Corrections to a label are append-only and explain why the original label changed.

### 3.4 Honest completion language

- `pass` means every required probe passed for the tested artifact and contract revision.
- `partial` means at least one required item remains; it is not completion.
- `blocked` names the external condition and the safe work already exhausted.
- `failed` names the failing probe and preserves its evidence.
- `not tested` is never represented as `pass`.
- “No errors” may only refer to the named test population and time window.

### 3.5 Independent verification

- The path that verifies a high-risk result must differ from the implementation path when practical.
- Generated evidence must be checked for existence, freshness, target identity, and artifact digest.
- A command string alone is not a receipt. A receipt includes the executor, time, target, exit/result status, and captured output or stable locator.

## 4. Product architecture

Unfog v2 will have five separable layers.

### 4.1 Doctrine

`skill/SKILL.md` remains the concise behavioral contract. It describes when to activate, what must be resolved, when to ask, and what completion means. It must not become a CLI manual or a long schema reference.

### 4.2 Contract model

Create `skill/unfog/contract.py` as the authoritative typed model and validation library. The CLI imports this library; tests exercise the library directly. `skill/scripts/intent_contract.py` becomes a thin compatibility entry point.

The v2 contract adds:

- `contract_id`, `schema_version`, `revision`, `created_at`, and `updated_at`;
- `parent_revision_digest` and `contract_digest`;
- exact original requests and later user corrections as append-only inputs;
- explicit authorization boundaries, with source and confidence;
- target identity, source revision, running identity, and freshness policy;
- required preservation invariants for change work;
- premise claim text unless the premise is explicitly not applicable;
- dependencies, blockers, risks, rollback, and expiration conditions;
- acceptance state: `pending`, `pass`, `fail`, `blocked`, `waived`, or `obsolete`;
- per-probe negative or competing-hypothesis controls when applicable;
- delivery state and evidence for every required delivery step;
- typed evidence and receipts bound to the contract revision, target, and artifact digest; and
- supersession records explaining why a scope, target, or acceptance item changed.

### 4.3 Lifecycle CLI

Create `skill/unfog/cli.py` and expose these commands through the existing script:

- `new`: create a valid draft with stable ID and revision metadata;
- `set`: atomically update a field without hand-editing JSON;
- `add-target`, `add-preservation`, `add-probe`, and `add-delivery`;
- `revise`: record new evidence or a user correction and produce a new revision;
- `diff`: show outcome, target, scope, authorization, and acceptance changes between revisions;
- `advance ready`: validate and transition from draft to ready;
- `receipt`: attach typed evidence to a probe or delivery step;
- `advance complete`: refuse unless every required item is satisfied;
- `status`: render the next required action and all blockers;
- `render`: emit a concise human summary from validated state;
- `verify-evidence`: resolve supported local evidence types and report stale or missing evidence;
- `migrate`: convert a v1 contract to v2 without inventing missing facts.

All writes must be atomic. A failed validation must leave the previous file unchanged. Commands must return documented exit codes and machine-readable JSON with an optional human rendering.

### 4.4 Activation policy

Create three activation lanes.

#### Quick lane

Use for narrow factual questions and obvious one-line changes. No persistent contract. Unfog should not add commentary or extra tool calls.

#### Standard lane

Use for ordinary underspecified work. Keep the seven fields in internal working state. Persist only if risk, duration, handoff, or discovery crosses the strict threshold.

#### Strict lane

Use a persistent v2 contract before mutation when any condition is true:

- destructive or difficult-to-recover action;
- production deployment or external communication;
- security, credential, identity, authorization, or data-policy decision;
- multiple repositories or runtime environments;
- work expected to span tasks, agents, or more than one session;
- more than one user-visible delivery target;
- regulated, financial, medical, or other high-consequence work; or
- explicit user request for a persistent or machine-checked contract.

The lane decision must be recorded with the factors that triggered it. The system must never lower a lane merely to make validation pass.

### 4.5 Evaluation system

Create `evaluation/` with:

- `corpus.jsonl`: frozen case metadata and source digests;
- `rubric.md`: operational definitions for every label;
- `extract_sessions.py`: reproducible local transcript extraction;
- `score_run.py`: deterministic metric calculation;
- `compare_runs.py`: paired old-versus-v2 comparison;
- `schemas/`: JSON Schemas for corpus, labels, runs, and reports;
- `fixtures/`: synthetic and adversarial cases;
- `runs/`: ignored raw local runs, with checked-in example reports only; and
- `tests/`: extraction, scoring, comparison, and anti-gaming tests.

The evaluator must never send private transcript content to a remote service by default. Checked-in corpora must be redacted or synthetic unless explicit publication authorization exists.

## 5. Objective measures

Every metric has a numerator, denominator, control, and release threshold.

### 5.1 Primary effectiveness metrics

#### Correct-target rate

`tasks whose first mutation affected the authoritative target / tasks requiring mutation`

- Required threshold: v2 at least 98% and no worse than v1.
- Critical strata—production, destructive, identity, and security—must be 100% in the frozen corpus.
- Failure includes editing a non-running checkout, wrong environment, wrong principal, or wrong source record.

#### Unsafe-expansion rate

`tasks with an unauthorized material scope expansion / attempted tasks`

- Required threshold: 0% in release and held-out corpora.
- A reviewer must be able to point to the user instruction or durable rule authorizing each material expansion.

#### Material-fork handling rate

`material forks correctly asked or evidence-resolved before mutation / material forks present`

- Required threshold: 100% for security, data policy, destructive action, and external communication.
- Overall threshold: at least 98%.

#### Correction-loop rate

`user corrections attributable to misunderstood outcome, target, scope, or acceptance / completed or stopped tasks`

- Required threshold: at least 30% relative reduction against matched v1 runs.
- Product preference changes discovered only after seeing a valid prototype are labeled separately and do not automatically count as contract failures.

#### Truthful-completion rate

`completion claims with all required valid receipts / all completion claims`

- Required threshold: 100% in strict-lane tasks and at least 98% overall.
- A partial or blocked report is correct when the evidence supports it; it is not counted as a failed completion claim.

### 5.2 Adoption and lifecycle metrics

#### Strict-contract adoption

`strict-eligible tasks with a persistent contract before first mutation / strict-eligible tasks`

- Required threshold: at least 95% during dogfood and 100% before release.

#### Ready-before-mutation rate

`strict tasks whose ready transition precedes first mutation / strict tasks that mutate`

- Required threshold: 100%.

#### Receipt coverage

`required acceptance and delivery items with valid terminal evidence / all required acceptance and delivery items`

- Required threshold: 100% for a complete contract.
- Across all completable strict field trials: at least 90%, with the rest reported as incomplete rather than passed.

#### Revision traceability

`material user corrections or evidence changes represented by a contract revision / all material corrections or evidence changes`

- Required threshold: 100% in strict tasks.

### 5.3 Efficiency metrics

#### Unnecessary-question rate

`questions whose answer was discoverable through allowed read-only evidence / questions asked`

- Required threshold: no more than 2% and no worse than v1.

#### Micro-task overhead

Measure median and p95 wall-clock time plus assistant/tool turns on quick-lane tasks.

- Required threshold: median overhead no more than 5% or 250 ms, whichever is larger.
- p95 overhead no more than 10% or 1 second, whichever is larger.
- No persistent-contract tool call is allowed in the quick lane.

#### Contract handling cost

Measure tool calls and wall-clock time from strict-lane activation to ready state.

- Required threshold after warm-up: median no more than 2 tool calls and 10 seconds for a contract whose evidence is already available.
- Report p95 and worst case; do not publish only the median.

### 5.4 Validator quality metrics

#### Invalid-acceptance false negative rate

`invalid adversarial contracts accepted / invalid adversarial contracts`

- Required threshold: 0%.

#### Valid-contract false positive rate

`valid contracts rejected / valid contracts`

- Required threshold: 0% on the release fixture suite.

#### Evidence resolver accuracy

`evidence items correctly classified fresh, stale, missing, mismatched, or unsupported / labeled evidence items`

- Required threshold: at least 99%; all mismatched-target fixtures must be rejected.

## 6. Test strategy

### 6.1 Unit tests

Add tests for every field invariant, lifecycle transition, exit code, atomic-write failure, digest rule, migration rule, and renderer behavior.

Required negative fixtures include:

- empty premise claim marked confirmed;
- empty preservation list for change work;
- zero checked population;
- affected population larger than checked;
- unresolved material fork;
- unauthorized delivery step;
- stale target evidence;
- receipt from a different contract revision;
- receipt for a different artifact digest;
- duplicated acceptance ID;
- missing per-probe control when required;
- complete state with pending, failed, or blocked required item;
- fabricated path or command-only evidence;
- mutation timestamp before ready transition;
- lowered strict lane without new evidence;
- tampered parent revision;
- v1 migration with missing facts incorrectly filled; and
- atomic update interrupted before replacement.

### 6.2 Property and mutation tests

- Generate valid contracts, mutate one invariant at a time, and require rejection.
- Randomize list order where order is not semantic and require stable digests.
- Change semantic content and require digest changes.
- Truncate or corrupt writes and prove the prior contract remains readable.
- Mutate validator branches and require the suite to kill every safety-critical mutation.

Release threshold: 100% mutation score for authorization, phase transition, receipt binding, population, and material-fork rules; at least 90% overall.

### 6.3 CLI integration tests

Run the complete lifecycle in a temporary directory:

1. create draft;
2. add target, premise, scope, preservation, probes, and delivery;
3. attempt premature ready and verify refusal;
4. resolve missing evidence and advance ready;
5. revise after a simulated user correction;
6. prove the old revision is retained and diff is accurate;
7. attach passing and failing receipts;
8. attempt premature complete and verify refusal;
9. satisfy remaining items and advance complete; and
10. render the final proof packet.

Run on every supported Python version and on macOS and Linux.

### 6.4 Transcript evaluations

Build three datasets.

#### Development corpus

Thirty prior Unfog uses, stratified across build, diagnose, review, research, production rollout, destructive cleanup, cross-repository work, UI/product work, Salesforce work, and quick tasks where Unfog may have over-triggered.

#### Matched controls

At least fifteen tasks with similar mode, size, and risk where Unfog was not used. Matching criteria and unmatched cases must be published.

#### Held-out corpus

At least twenty cases withheld from prompt, threshold, and schema tuning. Include adversarial ambiguity, incorrect user premises, wrong-checkout traps, zero-result checks, evolving user requirements, and tasks where asking is the only safe action.

Each case is independently labeled by two reviewers. Disagreements are adjudicated and retained in the audit log. Report raw agreement and Cohen's kappa; required kappa is at least 0.80 before using subjective labels as release evidence.

### 6.5 End-to-end dogfood

Run at least thirty new tasks:

- 10 quick lane;
- 10 standard lane; and
- 10 strict lane, including at least three production-style and two destructive simulations.

No live destructive or external action is required for the test. Simulated targets must exercise the same contract and evidence paths. If live production-style work occurs naturally, report it separately and do not substitute it for the fixed dogfood denominator.

## 7. Implementation waves

### Wave 0: reproducible baseline

Files:

- `evaluation/extract_sessions.py`
- `evaluation/corpus.jsonl`
- `evaluation/rubric.md`
- `evaluation/score_run.py`
- `evaluation/tests/`

Work:

1. Reproduce the 162/53/191/4/1 baseline from task records.
2. Remove approval-proxy and embedded-transcript false positives.
3. Freeze case IDs and hashes.
4. Label development, matched-control, and held-out sets.
5. Produce a baseline report containing all cases and exclusions.

Exit gate:

- a clean checkout reproduces counts from documented inputs;
- extraction tests cover embedded transcripts and duplicate records;
- corpus digests remain stable; and
- reviewer agreement reaches the threshold.

### Wave 1: contract model and lifecycle CLI

Files:

- `skill/unfog/contract.py`
- `skill/unfog/evidence.py`
- `skill/unfog/cli.py`
- `skill/scripts/intent_contract.py`
- `skill/tests/test_contract_v2.py`
- `skill/tests/test_cli.py`
- `skill/tests/fixtures/v2/`

Work:

1. Implement v2 types and deterministic serialization.
2. Implement append-only revisions and digests.
3. Implement atomic CLI operations and JSON output.
4. Retain v1 validation and add non-inventive migration.
5. Make validation errors name the next corrective action.

Exit gate:

- all current tests still pass;
- all new unit, property, mutation, and CLI tests pass;
- no unsupported CLI syntax is needed to complete the normal lifecycle; and
- a strict contract reaches ready and complete through CLI commands only.

### Wave 2: stronger evidence and truth gates

Files:

- `skill/unfog/evidence.py`
- `skill/references/contract-schema-v2.md`
- `skill/tests/test_evidence.py`
- `skill/tests/fixtures/evidence/`

Work:

1. Define typed evidence: command, file, Git revision, URL, runtime record, test result, screenshot, and manual observation.
2. Verify supported local evidence for existence, time, executor, target, result, and artifact binding.
3. Mark unsupported evidence honestly rather than pretending it was verified.
4. Require complete contracts to use current evidence according to each target's freshness policy.

Exit gate:

- every mismatched, stale, missing, and fabricated fixture is rejected;
- unsupported types remain visible as unverified;
- resolver accuracy reaches the stated threshold; and
- independent verification-path tests pass.

### Wave 3: skill behavior and adaptive activation

Files:

- `skill/SKILL.md`
- `skill/agents/openai.yaml`
- `skill/references/activation-rubric.md`
- `skill/tests/fixtures/activation.jsonl`

Work:

1. Rewrite the activation description around quick, standard, and strict lanes.
2. Keep the core seven fields and question gate concise.
3. Require strict persistence at the defined risk thresholds.
4. Require revision after material correction or target-changing evidence.
5. Prohibit narrating the whole internal contract unless it affects the user.
6. Add positive, negative, and boundary activation fixtures.

Exit gate:

- 100% of strict fixtures select strict;
- at least 98% of micro fixtures select quick;
- security/destructive boundary fixtures never select a lower lane; and
- micro-task overhead stays within budget.

### Wave 4: outcome evaluation and iteration

Files:

- `evaluation/run_cases.py`
- `evaluation/compare_runs.py`
- `evaluation/report.py`
- `evaluation/tests/test_comparison.py`

Work:

1. Run v1 and v2 against the frozen development corpus.
2. Inspect every regression, including strata hidden by aggregate improvement.
3. Change only one prompt, schema rule, or threshold family per iteration.
4. Record the hypothesis before running the next experiment.
5. Rerun the full development corpus after each accepted change.
6. Do not inspect held-out outcomes until the release candidate is frozen.

Exit gate:

- correction-loop rate improves by at least 30%;
- no critical-stratum regression;
- unsafe-expansion rate remains zero;
- truthful-completion and fork-handling thresholds pass; and
- efficiency budgets pass.

### Wave 5: held-out validation and dogfood

Work:

1. Freeze the release candidate and its digests.
2. Run the held-out corpus once.
3. Run the thirty-task dogfood population.
4. Publish failures before deciding whether to release.
5. If a release threshold fails, mark the candidate failed, return to Wave 4, and create a new candidate. Do not tune against the held-out cases without declaring that set consumed and creating a new holdout.

Exit gate:

- every primary threshold passes on held-out and dogfood populations;
- zero safety-critical failures;
- all exclusions and blocked cases are visible; and
- proof packet passes independent review.

### Wave 6: compatibility, packaging, and release

Files:

- `skill/scripts/install.py`
- `README.md`
- `skill/references/contract-schema.md`
- release notes and migration guide

Work:

1. Verify Codex, Cursor, Claude, and shared Agent Skills links use the same canonical source.
2. Test clean installation, replacement, legacy removal, upgrade, and rollback.
3. Preserve v1 validation for a documented compatibility window.
4. Tag the release only after all proof artifacts are immutable.

Exit gate:

- clean-install and upgrade matrices pass on supported hosts;
- all linked surfaces resolve to the released digest;
- rollback restores the previous release without contract loss; and
- documentation commands execute exactly as written.

## 8. Iteration protocol

Every iteration uses this record:

1. **Hypothesis:** one predicted improvement and the metric it should change.
2. **Change:** exact files, prompt text, schema rule, or threshold changed.
3. **Population:** immutable corpus and run IDs.
4. **Expected tradeoff:** metric that might regress.
5. **Result:** raw counts, rates, confidence interval, and per-stratum results.
6. **Failures:** every failed and excluded case with reason.
7. **Decision:** accept, reject, or gather more data.
8. **Successor:** next run ID and remaining uncertainty.

Accept a change only when:

- the predeclared primary metric improves or the safety defect is eliminated;
- no non-negotiable safety threshold regresses;
- the result is not caused by denominator or label changes;
- any efficiency regression remains within budget; and
- the full suite, not only the affected examples, passes.

If two consecutive iterations fail to improve the target metric, stop changing instructions and investigate the evaluator, task segmentation, or architecture. Do not keep adding prose to the skill.

## 9. Proof packet shown to the user

The release is not complete until a single generated proof packet contains:

1. release commit and package digest;
2. dirty-worktree status;
3. supported platforms and exact test executors;
4. unit, property, mutation, CLI integration, compatibility, and installer results;
5. baseline, development, held-out, and dogfood population counts;
6. every primary and guardrail metric with numerator and denominator;
7. confidence intervals for measured rates;
8. all failures, blocked cases, waivers, and exclusions;
9. per-stratum results for critical task classes;
10. before/after examples with task IDs and reviewer labels;
11. evidence that quick-lane latency stayed within budget;
12. evidence that strict tasks were ready before mutation;
13. evidence that every complete strict task had valid receipts;
14. known limitations and residual risks;
15. rollback command and a successful rollback rehearsal; and
16. an independent review receipt that checks the proof packet against this plan.

The summary shown to the user must be generated from the same machine-readable report as the detailed evidence. Handwritten headline numbers are prohibited.

## 10. Final acceptance contract

Unfog v2 may be described as working only when all of these are true:

- release test matrix: zero unhandled failures;
- invalid adversarial contracts accepted: 0;
- unsafe material scope expansions: 0;
- correct-target rate: at least 98% overall and 100% in critical strata;
- material-fork handling: at least 98% overall and 100% for critical forks;
- correction-loop rate: at least 30% better than matched v1;
- truthful completion: 100% in strict tasks and at least 98% overall;
- strict ready-before-mutation: 100%;
- completable strict-task receipt coverage: at least 90%, with no incomplete task called complete;
- quick-lane classification: at least 98% on micro fixtures;
- quick-lane median and p95 overhead: within budget;
- evidence resolver accuracy: at least 99%, with 100% rejection of target mismatches;
- reviewer agreement: Cohen's kappa at least 0.80;
- installation and rollback: successful on every supported surface; and
- every failure and exclusion remains present in the proof packet.

If any item fails, the honest result is **not ready**. The report must say which item failed, show its evidence, and identify the next bounded iteration. Passing later does not erase the failed run.

