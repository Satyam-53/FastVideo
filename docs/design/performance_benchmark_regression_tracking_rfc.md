# RFC: Performance Benchmark Regression Tracking Redesign

## Status

Draft.

## Context

FastVideo currently tracks inference performance through pytest benchmark runs,
normalized JSON records, a rolling Hugging Face baseline, per-GPU static
thresholds, and a Plotly dashboard. The current contributor documentation is
published at:

https://haoailab.com/FastVideo/contributing/performance_benchmarks/

The existing system is useful, but its comparison key is too coarse. Recent
records are grouped primarily by model/benchmark ID and GPU type. That means
unrelated changes can be treated as comparable:

- PyTorch, CUDA, container, or attention library upgrades.
- Different inference step counts or schedulers.
- Different precision or attention backend settings.
- Different prompt sets, output dimensions, or distributed layouts.
- Intentional fast variants that change the quality target.

The result is ambiguity: a reported "regression" may be a legitimate runtime
cohort shift, while a real regression may be hidden by a drifting baseline.

This RFC compares two redesign proposals and recommends a staged path that
fixes the comparability problem first, without taking on a full benchmark
database rewrite before FastVideo needs it.

## Inputs Compared

- `redesign_plan.md`: focused redesign around explicit comparability identity,
  manual calibration, metric-specific thresholds, and quality-aware variant
  promotion.
- `redesign_plan2.md`: larger "performance benchmark database" design with
  producer/store/consumer separation, Parquet/DuckDB storage, JSON sidecars,
  curated baseline sidecars, multiple execution modes, query-time status
  derivation, local upload policy, auto-seeding, cache fallback, and detailed
  operational procedures.

## Recommendation

Adopt the direction of `redesign_plan.md` as the first implementation phase.
Borrow selected rigor from `redesign_plan2.md`, especially:

- The split between `benchmark_version` and `result_schema_version`.
- Deterministic recipe fingerprint canonicalization rules.
- Explicit execution mode detection.
- Privacy guidance for local machine metadata.
- Clear curation semantics for promoted and excluded baseline records.
- Tests for identity, fingerprinting, recipe mismatch, calibration, and
  regression detection.

Defer the full database architecture from `redesign_plan2.md` until FastVideo
has enough workloads, hardware tiers, and dashboard/query pressure to justify
it.

In short: ship a v1++ comparator and schema upgrade first. Do not make Parquet,
DuckDB, sidecar curation, local public uploads, and query-time status
rederivation prerequisites for fixing today's regression tracking.

## Goals

- Compare only benchmark results that are actually comparable.
- Separate code regressions from runtime, hardware, recipe, and quality-target
  changes.
- Preserve the current low-friction CI flow and HF JSON storage where possible.
- Support scheduled-main benchmarks as the source of truth.
- Make new runtime or hardware cohorts explicit calibration events.
- Track component-level regressions, not only end-to-end latency.
- Keep the design implementable in small PRs.

## Non-Goals

- Replacing the current HF JSON store with a full Parquet database in the first
  phase.
- Building a generic storage abstraction before there is a second backend.
- Publishing local developer machine identity metadata to a public dataset.
- Automatically deciding whether a faster recipe is quality-equivalent.
- Migrating legacy benchmark history into the new identity model immediately.

## Proposed Design

### Comparable Identity

Two benchmark records are comparable only when all of these fields match:

```text
workload_id
variant_id
benchmark_version
hardware_profile_id
software_profile_id
recipe_fingerprint
```

Definitions:

- `workload_id`: stable benchmark family, for example `wan-t2v-1.3b`.
- `variant_id`: intentional recipe family, for example `canonical`,
  `fast-4step`, `flash-attn`, or `torch-sdpa`.
- `benchmark_version`: version of the measurement protocol and comparison
  policy.
- `hardware_profile_id`: normalized hardware cohort.
- `software_profile_id`: intentionally versioned runtime cohort.
- `recipe_fingerprint`: deterministic hash of inference settings that affect
  comparability.

The full environment should also be stored through `environment_fingerprint`,
but it should be audit metadata, not part of the default comparison key.

### Statuses

The comparator should report:

- `PASS`: comparable baseline exists and no gated metric regressed.
- `REGRESSION`: comparable baseline exists and at least one gated metric
  regressed.
- `CALIBRATION_NEEDED`: no comparable baseline exists for this identity.
- `RECIPE_MISMATCH`: same `variant_id` was used with a different
  `recipe_fingerprint`.
- `INFRA_ERROR`: benchmark infrastructure failed.
- `QUALITY_BLOCKED`: a candidate variant cannot be promoted because quality
  evidence is missing.

For CI gating:

- `REGRESSION` fails.
- `RECIPE_MISMATCH` fails unless the change intentionally creates a new
  variant.
- `CALIBRATION_NEEDED` should be visible but should not silently seed a passing
  baseline.
- `QUALITY_BLOCKED` blocks variant promotion, not normal regression checking.

### Recipe Fingerprint

The recipe fingerprint should include fields that change what was measured:

- Model path and resolved revision.
- Pipeline or preset name/version.
- Prompt and negative prompt digests.
- Height, width, frame count, and FPS.
- Seed.
- Number of inference steps.
- Scheduler settings.
- Guidance scale and embedded CFG scale.
- Attention backend.
- Sequence/tensor parallel size and GPU count.
- Text encoder, DiT, and VAE precision.
- VAE tiling, sequence parallel, and offload settings.
- Output type.
- Benchmark-specific overrides.

Use deterministic canonical JSON before hashing. Borrow from
`redesign_plan2.md` here: sorted keys, stable optional-field handling, stable
path/model revision normalization, SHA-256, and tests proving stable hashes for
semantically equivalent config forms.

### Software Cohorts

Runtime changes can affect performance without implying a FastVideo regression.
The `software_profile_id` should include major performance-affecting runtime
fields:

- Python major/minor.
- PyTorch major/minor.
- CUDA runtime major/minor.
- Triton major/minor when relevant.
- FlashAttention, SageAttention, or xFormers major/minor when used.
- Container performance profile version when available.

Patch versions, full package lists, driver details, OS/kernel, and image
digests belong in `environment_fingerprint` unless they prove noisy enough to
require a new cohort.

When `software_profile_id` changes, the comparator should report
`CALIBRATION_NEEDED` until maintainers seed or promote reviewed scheduled-main
records.

### Hardware Cohorts

The `hardware_profile_id` should normalize:

- GPU name.
- GPU count.
- GPU memory size.
- Relevant interconnect or machine class when stable and meaningful.

For multi-GPU benchmarks, records should preserve rank-level memory and timing
where possible, and gated aggregate metrics should use rank-max behavior when
that is the safe interpretation.

### Metrics

Keep end-to-end latency visible, but do not make it the only primary signal.

Primary gated metrics:

- `pipeline_total_s`
- `text_encode_s`
- `denoise_total_s`
- `denoise_step_mean_s`
- `denoise_step_p50_s`
- `denoise_step_p90_s`
- `transformer_forward_mean_s`
- `scheduler_step_mean_s`
- `vae_decode_s`
- `postprocess_cpu_s`
- `peak_memory_mb`
- `throughput_fps`

Informational metrics:

- `request_total_s`
- video write/export time
- generated artifact size
- individual run timings
- per-rank details

Canonical inference compute benchmarks should prefer `save_video=false` so
video encoding and filesystem noise do not dominate regression signals.

### Baseline Policy

For each exact comparable identity:

- Use scheduled-main records only for canonical baselines.
- Keep failed, calibration, and mismatch records for audit.
- Exclude failed records from baseline medians.
- Compare current results against a rolling baseline of recent successful
  scheduled-main records.
- Keep a promoted baseline as an explicit maintainer-approved reference.
- Alert if a run regresses against either rolling or promoted baseline.
- Do not let `CALIBRATION_NEEDED` records automatically become baseline seeds
  in Phase 1.

This intentionally follows the more conservative behavior from
`redesign_plan.md`. `redesign_plan2.md` contains an auto-seed path after enough
new-cohort rows accumulate; that is convenient, but it weakens the guardrail
that the existing `reseed-performance-baseline` workflow was created to
provide.

### Thresholds

Use metric-specific thresholds with both percent and absolute delta floors.

Suggested defaults:

- 5% for core compute metrics.
- 5% for peak memory.
- 8-10% for noisy end-to-end or request-level metrics.
- Absolute floors such as 250 ms for stage totals, 50 ms for per-step metrics,
  and a reasonable MB floor for memory.

A metric regresses only when:

```text
percent_delta > threshold_percent
absolute_delta > threshold_absolute
```

The adaptive dispersion-based thresholds in `redesign_plan2.md` are promising,
but should be deferred until there is enough history to validate false-positive
and false-negative behavior.

### Variants And Quality

Same-recipe implementation optimizations stay in the same variant. Quality or
recipe changes get a new variant.

Examples:

| Change | Variant behavior |
|---|---|
| Faster implementation of the same attention backend | Same variant |
| Reduced inference steps | New variant |
| Different default attention backend | New variant |
| Different precision recipe | New variant |
| DMD or distilled recipe | New variant |

New quality-affecting variants should start as candidates:

```text
candidate -> calibrating -> canonical
candidate -> deprecated
```

Promotion to canonical requires:

- Quality evidence, such as SSIM, latent similarity, existing evaluation
  metrics, or reviewed generated artifacts.
- Stable scheduled-main performance records.
- Explicit maintainer approval.

Missing quality evidence should produce `QUALITY_BLOCKED` for promotion.

## What Looks Overengineered In Plan 2

Several parts of `redesign_plan2.md` are valuable eventually, but too heavy for
the immediate problem:

1. Full Parquet/DuckDB store as Phase 1.
   FastVideo currently has a small number of tracked workloads. A per-run JSON
   layout with richer identity fields is enough to fix incorrect comparisons.
   Parquet becomes attractive when dashboard/query performance or concurrent
   write volume becomes painful.

2. Producer/store/consumer architecture as an explicit platform.
   The separation is conceptually clean, but formalizing it now risks turning a
   comparator fix into an infrastructure project.

3. JSON sidecars plus Parquet row hashes.
   Sidecars are useful when records become too nested for tabular storage. In
   Phase 1, a single normalized JSON record can carry identity, metrics,
   provenance, and audit metadata.

4. Per-identity curated baseline sidecar hierarchy.
   Curation is needed, but a simpler reviewed baseline seed file or reseed
   workflow extension is probably enough at first.

5. Local public uploads.
   Local read-only comparison is useful. Publishing local records to the public
   HF dataset adds privacy and policy complexity that is not necessary for
   scheduled-main regression tracking.

6. Auto-seeding new cohorts.
   Convenient, but risky. Runtime and hardware cohort changes are exactly where
   human review is most useful.

7. Adaptive sigma/MAD thresholding as an initial gate.
   Statistically appealing, but more complex to explain and tune than
   per-metric percent and absolute thresholds. It should come after enough
   historical data exists.

8. Detailed redaction, schema-skew, cache fallback, pending-upload, and
   operator alerting machinery.
   These are good production concerns, but they should not block the first
   comparability fix. Add them as failure modes become real.

9. Full quality lifecycle storage.
   A lightweight `quality_status` plus links to evidence is enough initially.
   A formal lifecycle sidecar can come later if FastVideo has many competing
   variants.

10. Storage backend generalization.
    There is one real backend today: Hugging Face. Avoid a storage abstraction
    until there are two real implementations.

## Phase 1 Implementation Plan

1. Extend benchmark configs with `workload_id`, `variant_id`,
   `benchmark_version`, recipe fields, and metric threshold policy.
2. Update `test_inference_performance.py` to emit v2 normalized JSON records
   with identity, recipe fingerprint, hardware/software profile IDs, runtime
   audit metadata, and component metrics.
3. Add deterministic recipe fingerprinting with unit tests.
4. Update `compare_baseline.py` to compare only exact identity matches.
5. Report `CALIBRATION_NEEDED` for missing hardware/software/recipe cohorts.
6. Report `RECIPE_MISMATCH` when a variant's recipe fingerprint changes.
7. Add promoted baseline support through the existing reseed workflow.
8. Add metric-specific threshold support.
9. Update `dashboard.py` to group by workload, variant, hardware profile,
   software profile, and benchmark version.
10. Keep legacy v1 JSON records readable for historical charts, but do not
    compare v2 records against v1 records by default.

## Phase 2 Candidates

Revisit the heavier parts of `redesign_plan2.md` when at least one of these is
true:

- FastVideo tracks five or more workloads across multiple hardware tiers.
- Dashboard queries become slow because JSON history is too large.
- Concurrent scheduled-main writes become unreliable.
- Maintainers need ad-hoc SQL analysis across many variants and cohorts.
- Baseline curation becomes too complex for the reseed workflow.

Phase 2 can then introduce:

- Hive-partitioned Parquet.
- DuckDB/pandas query consumers.
- Sidecar JSON for deep audit payloads.
- Curated promoted/excluded baseline records.
- Local opt-in uploads with privacy controls.
- Adaptive thresholding derived from cohort variance.
- Stronger operator workflows for redaction, schema skew, and upload recovery.

## Migration Plan

- Treat current records as schema v1.
- Start schema v2 with the existing Wan benchmark:
  - `workload_id = wan-t2v-1.3b`
  - `variant_id = canonical`
  - `benchmark_version = 1`
- Seed v2 baselines manually from reviewed scheduled-main artifacts.
- Keep v1 charts visible as historical context.
- Do not aggregate v1 and v2 records unless an explicit migration derives
  compatible v2 identities.

## Test Plan

Required tests:

- Same config produces the same recipe fingerprint.
- Semantically equivalent config forms produce the same recipe fingerprint
  where intended.
- Changing `num_inference_steps` changes the recipe fingerprint.
- Changing PyTorch/CUDA profile produces `CALIBRATION_NEEDED`.
- Same variant with changed recipe produces `RECIPE_MISMATCH`.
- Slower gated metric produces `REGRESSION`.
- Faster or unchanged metrics produce `PASS`.
- Candidate variants do not compare against canonical variants.
- Missing quality evidence blocks promotion.
- Dashboard grouping separates software cohorts and variants.
- Current Wan config writes valid v2 JSON with populated identity and metrics.

## Open Questions

1. Which runtime fields should be included in `software_profile_id` for the
   first implementation? The RFC recommends major/minor Python, PyTorch, CUDA,
   and relevant attention/kernel libraries.
2. Should the first promoted baseline compare against one reviewed record or a
   median of several reviewed scheduled-main records?
3. Should `CALIBRATION_NEEDED` fail scheduled-main builds, or report visibly
   without failing? This RFC recommends visible non-failure.
4. Which metrics should be gated for the first Wan benchmark versus recorded
   only as informational?
5. Where should quality evidence for candidate variants live initially: test
   references, benchmark config metadata, or a small reviewed manifest?

## Decision Summary

Choose the conservative implementation path:

- Fix comparability first.
- Keep storage simple.
- Require human-reviewed seeding for new cohorts.
- Use explicit variants for quality-changing recipes.
- Borrow Plan 2's rigor where it clarifies identity and schema behavior.
- Defer Plan 2's database/platform machinery until scale makes it worth the
  maintenance cost.
