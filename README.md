# QPPE: Certified Query Steering

Reproducibility repository for **"Certified Query Steering: Finite-Sample
Risk Control for Learned Plan Selection"**, submitted to *The VLDB Journal*.

## What this is

QPPE wraps learned PostgreSQL query-plan steering in a finite-sample
**conformal risk certificate**. A machine-learned model scores candidate
plans for regression risk and improvement potential, but the system is
only allowed to steer a query to a non-default plan when a calibration
procedure certifies that doing so keeps the probability of a *materially
severe regression* below a pre-declared bound (by default: more than 2x
slower **and** more than 1 second slower than the default plan, or a
timeout). If the calibration record can't support that guarantee, the
system abstains and runs the default plan — and we treat that abstention
as a correct, desirable outcome, not a failure.

Every script below is part of one continuous pipeline: later scripts
import earlier ones, and every number, table, and figure in the paper
traces back to one of them. Run them in order; each is a single command
and resumable if interrupted.

## Requirements

- Python 3.11+ (`py` on Windows, `python3` on Linux/macOS)
- PostgreSQL 18.x running locally
- DuckDB and MySQL 8.x, for the portability chapter only (Steps 9a/9b)
- `pip install -r requirements.txt`

Standard invocation pattern throughout: `py stepN_name.py --user postgres
--password <yours>`

---

## The pipeline, step by step

### Phase 0 — Environment setup

**`step1_check_env.py`** and **`step1b_remaining_checks.py`**
Verify the PostgreSQL install actually supports what the rest of the
pipeline needs before any real work starts: all 13 planner-steering GUCs
respond correctly, `SET LOCAL` scoping behaves as expected (so hint sets
don't leak between queries), and `EXPLAIN (FORMAT JSON)` output parses
cleanly. Cheap to run, and catches environment problems early rather than
three hours into a corpus collection run.

### Phase 1 — TPC-H corpus construction

**`step2_inspect_tpch.py`**
Inspects an existing TPC-H installation. This is where we discovered the
original database was scale factor 0.04, not 1 — mislabeled data that had
been silently corrupting an earlier draft of this project's results.

**`step2b_build_sf1.py`**
Builds a properly spec-verified TPC-H SF1 database from scratch via
DuckDB's built-in `dbgen`, then checks the result against the known-correct
row counts (`lineitem`: 6,001,215 rows) rather than trusting the generator
blindly.

**`step2c_configure.py`**
Applies and freezes the PostgreSQL configuration used for every timing
measurement in the paper (`shared_buffers=7GB`, `work_mem=64MB`, JIT off,
parallelism capped) — held constant so that timing differences reflect the
steering decision, not a moving configuration target.

**`step3a_collect_plans.py`**
For every query template, collects a candidate plan under each of 12 hint
sets (combinations of planner GUCs like `enable_nestloop`,
`enable_hashjoin`, `enable_seqscan`), deduplicated by a structural hash of
the plan tree.

**`step3b_execute_plans.py`**
Executes every *distinct* plan: one untimed warm-up run, then three timed
runs, with an adaptive timeout (3x the default plan's own runtime, floor
10s, cap 120s) so that a handful of catastrophic candidates can't stall
the whole collection run for hours.

**`step3c_expand_corpus.py`**
Expands the corpus to its final form: 20 templates, 89 query instances.
This script also exports the shared `TEMPLATES`, `HINT_SETS`, and plan
feature-extraction code that every later script imports — it's the
backbone the rest of the pipeline is built on.

### Phase 2 — Model and feature iteration

This phase is a record of what *didn't* work, kept because the dead ends
are informative:

**`step4_label_and_train.py`**
First attempt at a regression-risk classifier, using raw plan features
(estimated cost, row counts) directly. Overfits to template-specific
scale — a feature that means one thing for a simple 2-table join means
something completely different for a 6-way join, and the model can't
generalize across templates as a result.

**`step4b_model_v2.py`**
Fixes this with scale-invariant *ratio* and *delta* features (candidate
cost ÷ default cost, rather than either in isolation). This is the fix
that makes cross-template generalization possible at all.

**`step4c_two_head_policy.py`**
Splits the model into two heads: one scoring regression risk, one scoring
win probability — the architecture the final policy still uses.

**`step4d_two_head_expanded.py`**
Expands the training corpus to include censored (timed-out) plans,
explicitly labeled as forced regressions rather than silently dropped.

**`step4e_enriched_features.py`**
Adds node-level plan features — maximum nested-loop inner cardinality,
maximum intermediate-result blow-up, dominant-node cost share — which is
what finally lets the win-head recognize a good steering opportunity in a
template it never saw during training.

### Phase 3 — The conformal gate (first construction)

**`step5_conformal_gate.py`**
Introduces the actual certificate: a Clopper–Pearson-calibrated
threshold, the largest one whose upper confidence bound on the regression
rate stays under the target level. Runs the first two validity
experiments (E1: stationary workload, E2: shift to an unseen template).

**`step6_live_loop.py`**
The first attempt at a real, live, end-to-end run — not simulation.
Calibrating on a single 75/25 split turns out to be a mistake: the
calibration set doesn't contain enough clean examples, and the threshold
comes back as `t* = 0`. The system correctly refuses to steer anything,
which is the right behavior given insufficient evidence — but it's also
the first sign of a deeper sample-size problem the project spends the
next several steps chasing down.

**`step6b_cross_conformal.py`**
Fixes it with 5-fold cross-fitted calibration instead of a single split.
Result: `t* = 0.218`, and live on 17 fresh queries: 16 steered, 0
regressions, +23.6% net workload improvement. The headline TPC-H result,
at this stage of the construction.

### Phase 4 — The JOB benchmark and the valid construction

This phase is where the project's central theoretical contribution
actually gets discovered, through a sequence of failures that turn out to
be more informative than a clean success would have been.

**`step7a_setup_job.py`**
Loads the IMDB dataset and the 113 JOB (Join Order Benchmark) queries —
real, correlated, skewed data, unlike TPC-H's uniform synthetic
distributions.

**`step7b_job_pipeline.py`**
Collects and executes the JOB corpus: 113 queries x 12 hint sets. JOB is
dramatically harsher than TPC-H — individual default-plan failures up to
43x worse than the best alternative.

**`step7c_job_analysis.py`**
Runs E1/E2 on JOB and a new E3 (cross-benchmark transfer: calibrate on
TPC-H, deploy cold on JOB). The gate seals shut entirely — `t* = 0` on
JOB no matter how it's calibrated.

**`step7d_gate_diagnostics.py`**
Diagnoses why: the risk model's ranking is fine (AUC ~0.87), but the
*clean lowest-risk prefix* — how many of the safest-ranked candidates in
a row are actually safe — is only 2. Far too short to support any
threshold.

**`step7e_severity_gate.py`**
Tries relabeling the certified event as "severe" (>2x slowdown) rather
than any regression. Prefix improves to 7 — better, still not enough.

**`step7f_material_severity.py`**
Adds an absolute-damage floor to the event definition: severe means >2x
slowdown **and** >1 second of added latency. Prefix jumps to 32, and the
gate finally opens — but E1 reveals a new problem: the realized violation
rate exceeds its own target in 7 of 20 workload draws, when only about 2
would be expected. The naive calibration construction is
**anti-conservative** on this harsher benchmark.

**`step7g_policy_calibration.py`**
First attempt at the fix: calibrate the deployed policy directly, with
queries (not candidates) as the exchangeable unit. A bug in the
threshold-search order causes it to fail at the first grid point tried.

**`step7h_select_then_certify.py`**
A cleaner attempt, which fails for an entirely different, more
informative reason: the corpus is arithmetically too small to support a
held-out calibration split at all. This is where the paper's sample-size
floor theorem comes from directly — it isn't derived first and confirmed
later, it's discovered by hitting the wall in practice.

**`step7i_certified_live.py`**
The valid construction: cross-fitted, policy-level, query-unit
calibration, validated live on fresh queries on *both* benchmarks. TPC-H:
+15.0% net, 0 severe regressions. JOB: the gate correctly abstains on all
15 fresh queries rather than guess — exactly the behavior the sample-size
floor predicts, now demonstrated live rather than just derived.

### Phase 5 — Figures, portability, and the appendix

**`step8_make_figures.py`**
Regenerates all 8 figures in the paper directly from the logged
databases and pickled corpora — nothing in the figures is hand-edited.

**`step9a_duckdb_port.py`**
Ports the entire framework to DuckDB. Finding: DuckDB's default plans are
already close to optimal on this workload (~5% oracle headroom), so the
gate correctly certifies almost nothing — there's little unsafe headroom
to protect against in the first place.

**`step9b_mysql_port.py`**
Ports to MySQL. Different finding, same correct behavior: MySQL's default
plans are genuinely far from optimal, but the available steering knobs
(`optimizer_switch`) are too coarse to reach the better plans that likely
exist — so again, the gate mostly abstains, for a different underlying
reason than DuckDB.

**`step10_appendix.py`**
Recomputes every table and calibration in the paper's statistical
appendix directly from the primary logged data — the check that nothing
in the writeup drifted from what the scripts actually produced.

---

## Data artifacts

- **`qppe_duckdb_corpus.pkl`**, **`qppe_mysql_corpus.pkl`** — the
  collected plan/timing corpora for the two portability engines, included
  so that Section 7 of the paper is reproducible without re-running the
  (multi-hour) collection phase from scratch.
- **`figures/`** — all 8 paper figures, as both `.pdf` (used in the
  manuscript) and `.png`.
- **`job_queries/`** — the 113 JOB query text files.
- **`appendix.md`** — a worked example of `step10_appendix.py`'s output.

## Citation

```
Dilekh, T. Certified Query Steering: Finite-Sample Risk Control for
Learned Plan Selection. Submitted to The VLDB Journal, 2026.
```

## License

MIT License (see `LICENSE`).
