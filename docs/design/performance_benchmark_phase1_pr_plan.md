# Phase 1 PR Plan: Performance Benchmark Regression Tracking

## Purpose

This plan decomposes the Phase 1 redesign from
`performance_benchmark_regression_tracking_rfc.md` into reviewable PRs. The
goal is to fix benchmark comparability without turning the first iteration into
a storage-platform rewrite.

Each PR should be independently reviewable, have a narrow debug surface, and
avoid depending on unreleased behavior from later PRs unless explicitly noted.

## Current Baseline

The current implementation has three main pieces:

- `test_inference_performance.py` runs config-driven inference benchmarks and
  writes raw `perf_*.json` files.
- `compare_baseline.py` normalizes raw results, compares against the last 5
  successful HF records for `(model_id, gpu_type)`, writes Markdown summaries,
  and uploads normalized records on full main runs.
- `hf_store.py` syncs and uploads JSON records under
  `FastVideo/performance-tracking/<model_id>/`.

Phase 1 keeps this JSON/HF flow and upgrades the schema, comparison key, and
reporting behavior.

## PR Stack

### PR 1: Add v2 Identity Fields To Benchmark Configs

**Goal:** Make benchmark intent explicit before changing producer or comparator
behavior.

**Depends on:** None.

**Changes:**

- Add v2 metadata to `.buildkite/performance-benchmarks/tests/*.json`:
  - `workload_id`
  - `variant_id`
  - `benchmark_version`
  - optional `quality_status`, defaulting to `canonical` for the current Wan
    benchmark
- Keep existing `benchmark_id`, `model`, `init_kwargs`, `generation_kwargs`,
  `run_config`, and `thresholds` unchanged.
- Document that `benchmark_id` remains the legacy identifier during migration.

**Debug boundary:** If this PR breaks anything, the issue is config loading or
schema compatibility only. Runtime benchmark behavior should be unchanged.

**Tests/checks:**

- Existing performance config discovery still loads the Wan config.
- A lightweight config validation unit test can assert required v2 keys are
  present.

**Acceptance criteria:**

- Existing benchmark execution remains compatible.
- No comparator behavior changes yet.

### PR 2: Add Identity And Fingerprint Helpers

**Goal:** Introduce deterministic identity construction in isolated helper code.

**Depends on:** PR 1.

**Changes:**

- Add helper functions for:
  - recipe payload extraction from benchmark config
  - deterministic recipe fingerprint creation
  - hardware profile ID creation
  - software profile ID creation
  - environment fingerprint creation
- Use stable JSON canonicalization for the recipe fingerprint:
  sorted keys, compact separators, omitted unset optional fields, prompt
  digests instead of raw prompt text inside the hash payload.
- Keep helper output independent from HF storage and comparator decisions.

**Debug boundary:** Failures are limited to deterministic hashing and metadata
construction. No benchmark gate should change yet.

**Tests/checks:**

- Same config produces the same recipe fingerprint.
- Dict key order does not change the fingerprint.
- Prompt text changes the prompt digest and recipe fingerprint.
- `num_inference_steps` changes the recipe fingerprint.
- Hardware and software profile helpers return stable non-empty strings for
  mocked metadata.

**Acceptance criteria:**

- Helper functions are pure or easy to mock.
- No upload path or baseline comparison behavior changes yet.

### PR 3: Emit v2 Fields In Raw And Normalized JSON

**Goal:** Start producing v2-compatible records while preserving legacy readers.

**Depends on:** PR 2.

**Changes:**

- Update benchmark raw result JSON to include:
  - `result_schema_version`
  - `workload_id`
  - `variant_id`
  - `benchmark_version`
  - `recipe_fingerprint`
  - `hardware_profile_id`
  - `software_profile_id`
  - `environment_fingerprint`
  - `quality_status`
  - `recipe`, `hardware`, and `software` audit blocks
- Update normalization so normalized HF-ready records preserve both:
  - legacy fields: `model_id`, `gpu_type`, `latency`, `throughput`, `memory`,
    component timings, `success`
  - v2 fields listed above
- Keep writing normalized records to the existing `<model_id>/...json` layout.

**Debug boundary:** If this PR fails, inspect JSON serialization and
normalization. Baseline filtering should still use legacy `(model_id,
gpu_type)`.

**Tests/checks:**

- Normalizing a legacy raw record still works.
- Normalizing a v2 raw record preserves all v2 identity fields.
- Raw result schema includes expected v2 fields when helpers are mocked.

**Acceptance criteria:**

- Existing dashboard and comparator still work with old records.
- New records carry enough identity metadata for later PRs.

### PR 4: Add Identity-Aware Record Loading

**Goal:** Teach storage helpers to find comparable v2 records without changing
CI gates yet.

**Depends on:** PR 3.

**Changes:**

- Add a loader that filters records by exact v2 identity:
  `workload_id`, `variant_id`, `benchmark_version`, `hardware_profile_id`,
  `software_profile_id`, and `recipe_fingerprint`.
- Keep `load_records_for_model()` unchanged for legacy callers.
- Ensure legacy records without v2 fields are ignored by v2 identity loading.

**Debug boundary:** Problems are isolated to HF/local JSON filtering. No
regression status semantics change yet.

**Tests/checks:**

- Exact identity match returns only comparable records.
- Different recipe fingerprint is excluded.
- Different software profile is excluded.
- Legacy records are ignored by v2 loader but still readable by legacy loader.

**Acceptance criteria:**

- Both legacy and v2 loading paths can coexist.
- No storage layout migration is required.

### PR 5: Switch Comparator To v2 Identity With Calibration Status

**Goal:** Stop invalid comparisons by using exact identity for v2 records.

**Depends on:** PR 4.

**Changes:**

- In `compare_baseline.py`, use v2 identity loading when the current record has
  `result_schema_version >= 2`.
- Keep legacy comparison for v1 records.
- When no exact v2 baseline exists, report `CALIBRATION_NEEDED`.
- Do not upload calibration-needed records as successful baseline seeds unless
  the run is explicitly part of a reviewed reseed workflow.
- Update Markdown summary to show status per row:
  `PASS`, `REGRESSION`, or `CALIBRATION_NEEDED`.

**Debug boundary:** If false positives appear, inspect identity fields and
baseline record selection. Metric threshold logic should still be the existing
`PERF_MAX_REGRESSION` behavior.

**Tests/checks:**

- Matching v2 baseline produces normal pass/fail comparison.
- Missing v2 baseline reports `CALIBRATION_NEEDED`.
- v1 records still compare by legacy `(model_id, gpu_type)`.
- Calibration-needed rows are visible in Markdown summary.

**Acceptance criteria:**

- PyTorch/CUDA/hardware/recipe cohort changes no longer compare against old
  baselines.
- Existing local and PR workflows still produce a useful report.

### PR 6: Add Recipe Mismatch Detection

**Goal:** Make accidental recipe drift fail loudly instead of silently creating
new incomparable records.

**Depends on:** PR 5.

**Changes:**

- Detect records with the same:
  `workload_id`, `variant_id`, `benchmark_version`, `hardware_profile_id`, and
  `software_profile_id`, but a different `recipe_fingerprint`.
- Report `RECIPE_MISMATCH` with enough summary detail to identify the current
  and baseline fingerprints.
- Fail CI for `RECIPE_MISMATCH`.
- Keep genuine new variants separate by requiring `variant_id` changes for
  intentional quality/recipe changes.

**Debug boundary:** Problems are isolated to near-identity lookup and status
classification.

**Tests/checks:**

- Same variant with changed step count reports `RECIPE_MISMATCH`.
- New variant with changed step count reports `CALIBRATION_NEEDED`, not
  `RECIPE_MISMATCH`.
- Mismatch status fails comparator exit code.

**Acceptance criteria:**

- Accidental benchmark recipe edits are not mistaken for clean new cohorts.

### PR 7: Add Metric-Specific Threshold Policy

**Goal:** Replace the one-size-fits-all regression threshold with explicit
per-metric gates.

**Depends on:** PR 5. Can be reviewed before PR 6 if needed, but should merge
after PR 5.

**Changes:**

- Add per-metric threshold config with percent and absolute floors.
- Preserve `PERF_MAX_REGRESSION` as the default percent threshold when no
  per-metric value is configured.
- A metric regresses only if both percent and absolute deltas exceed the
  configured floors.
- Keep static pytest thresholds as broad fail-safes.

**Debug boundary:** If a gate looks wrong, inspect only metric direction,
baseline median, percent delta, and absolute delta. Identity selection has
already been handled by earlier PRs.

**Tests/checks:**

- Percent-only tiny absolute movement does not fail.
- Large absolute movement below percent floor does not fail.
- Both percent and absolute threshold breach fails.
- Throughput remains higher-is-better while latency/memory/timing are
  lower-is-better.

**Acceptance criteria:**

- Summary includes enough per-metric detail to explain why a metric failed.

### PR 8: Extend Promoted Baseline / Reseed Workflow For v2 Identity

**Goal:** Allow maintainers to intentionally seed or advance a reviewed v2
baseline.

**Depends on:** PR 5.

**Changes:**

- Update the reseed workflow to validate a single exact v2 identity tuple.
- Reject source JSON batches that mix workload, variant, benchmark version,
  hardware profile, software profile, or recipe fingerprint.
- Continue writing accepted records into the existing HF JSON layout.
- Document that manual reseed is the approved way to move a
  `CALIBRATION_NEEDED` cohort into comparable status.

**Debug boundary:** Problems are isolated to baseline curation and reviewed
record upload, not benchmark execution.

**Tests/checks:**

- Mixed identity reseed input is rejected.
- Consistent v2 identity reseed input is accepted.
- Legacy reseed behavior remains usable during migration.

**Acceptance criteria:**

- Maintainers have a clear path to seed new runtime/hardware cohorts.

### PR 9: Dashboard Grouping For v2 Identity

**Goal:** Make maintainer views match the new comparison model.

**Depends on:** PR 4. Best merged after PR 5 so statuses are meaningful.

**Changes:**

- Group dashboard plots by v2 identity when v2 fields are present:
  workload, variant, benchmark version, hardware profile, and software profile.
- Keep legacy grouping by `model_id` and `gpu_type` for v1 records.
- Show recipe fingerprint, quality status, and schema version in hover or table
  metadata.
- Keep skipped metric reporting.

**Debug boundary:** Dashboard bugs should not affect CI gate behavior.

**Tests/checks:**

- v2 records with different software profiles produce separate plot groups.
- v2 records with different variants produce separate plot groups.
- v1 records still render in legacy groups.

**Acceptance criteria:**

- Maintainers can visually distinguish runtime cohorts, variants, and legacy
  history.

### PR 10: Documentation And Rollout Guide

**Goal:** Make the new behavior understandable for CI users, maintainers, and
local developers.

**Depends on:** PRs 5-9.

**Changes:**

- Update `docs/contributing/performance_benchmarks.md` with:
  - v2 identity model
  - new statuses
  - calibration behavior
  - recipe mismatch behavior
  - local developer read-only expectations
  - reseed workflow for new cohorts
- Link to the RFC and this PR plan.
- Add a short migration note explaining v1 history and v2 records.

**Debug boundary:** Docs-only after behavior lands.

**Tests/checks:**

- Docs build/pre-commit as applicable.
- Commands in the quick start remain accurate.

**Acceptance criteria:**

- A contributor can understand why a run is `CALIBRATION_NEEDED` or
  `RECIPE_MISMATCH` without reading implementation code.

## Recommended Merge Order

1. PR 1: Config metadata.
2. PR 2: Identity/fingerprint helpers.
3. PR 3: v2 record emission.
4. PR 4: v2 identity loading.
5. PR 5: identity-aware comparator and calibration.
6. PR 6: recipe mismatch detection.
7. PR 7: metric-specific thresholds.
8. PR 8: v2 reseed workflow.
9. PR 9: dashboard grouping.
10. PR 10: docs and rollout guide.

PR 7 and PR 8 can be developed in parallel after PR 5. PR 9 can begin after
PR 4, but should merge after PR 5 so the dashboard matches comparator
semantics.

## Debugging Strategy

- If raw JSON is wrong, debug PRs 2-3.
- If comparable records are missing, debug PR 4.
- If statuses are wrong, debug PRs 5-6.
- If a metric fails unexpectedly, debug PR 7.
- If maintainers cannot seed a baseline, debug PR 8.
- If charts are confusing or merged incorrectly, debug PR 9.

This keeps each failure class tied to one small part of the stack.

## Cross-PR Compatibility Rules

- v1 records must remain readable until a separate migration is approved.
- v2 records must continue to include legacy normalized fields during Phase 1.
- The HF storage layout stays unchanged in Phase 1.
- Local runs remain read-only by default.
- Scheduled-main records remain the only source for official rolling baselines.
- `CALIBRATION_NEEDED` is visible but does not silently become a passing
  baseline.

## Phase 1 Completion Criteria

Phase 1 is complete when:

- Current Wan benchmark emits v2 identity metadata.
- Comparator uses exact v2 identity for v2 records.
- Runtime, hardware, and recipe changes no longer compare against old baselines.
- Accidental recipe drift reports `RECIPE_MISMATCH`.
- Missing exact baselines report `CALIBRATION_NEEDED`.
- Maintainers can seed reviewed v2 baselines.
- Dashboard separates variants and runtime/hardware cohorts.
- Docs explain the new statuses and migration behavior.
