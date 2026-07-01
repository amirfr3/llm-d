# Nightly E2E verification scripts

One Python file per scenario. Each script imports helpers from
[`verify_helpers.py`](verify_helpers.py), expresses its checks inline,
and adds any scenario-specific inspection (PVCs, EPP logs, etc.). No YAML
config files — the script is the single source of truth.

Verification runs between the harness `Run` step and `Teardown` in
`reusable-ci-nightly-benchmark.yaml` (llm-d-infra), so pods and PVCs are
still up and `kubectl` is configured against the test cluster.

## Layout

```
nightly-e2e-verification/
├── README.md
├── verify_helpers.py         ← shared helpers (library only; not run directly)
├── _template/
│   └── verify.py             ← copy this for a new scenario
└── tiered-prefix-cache/
    └── verify.py             ← inline metric checks + PVC inspection
```

## Wiring up a caller workflow

Point at the per-scenario script:

```yaml
uses: llm-d/llm-d-infra/.github/workflows/reusable-ci-nightly-benchmark.yaml@main
with:
  verify_script: .github/scripts/nightly-e2e-verification/tiered-prefix-cache/verify.py
```

If `verify_script` is empty, no verification runs.

## Adding a new scenario

```bash
cp -r _template <your-scenario>
$EDITOR <your-scenario>/verify.py
```

Then in `<your-scenario>/verify.py`:

1. **Inline checks.** Add `metrics.check_aggregated(...)` and/or
   `metrics.check_per_pod(...)` calls to the `checks` list. Aggregates and
   ops are listed below.
2. **Scenario-specific inspection (optional).** Add `print()`s and custom
   `v.Check(...)` objects — see [tiered-prefix-cache](tiered-prefix-cache/verify.py)
   for a worked example.
3. Point the caller workflow at the new path.

The script is the single source of truth — grep-friendly, commentable per
line, no separate config file to drift.

## API reference

```python
import verify_helpers as v

env         = v.workflow_env()                            # dict of env vars
exp_dirs    = v.find_results_dirs(env["workspace"])       # list of experiment subdirs
metrics     = v.MetricsSummary.load(exp_dirs[0])          # wraps metrics_summary.json
```

### `MetricsSummary`

Three views over `metrics_summary.json`:

```python
metrics.raw          # the entire loaded JSON (escape hatch)
metrics.aggregated   # {metric_name: {mean, max, p99, ...}}   ← _aggregated.metrics
metrics.per_pod      # {pod_name: {metric_name: {mean, max, ...}}}
```

Two check factories:

```python
# Reads metrics.aggregated[metric][aggregate], threshold-checks it.
metrics.check_aggregated("vllm:time_to_first_token_seconds", "p99", "<=", 2.0)
# → Check(name="vllm:time_to_first_token_seconds.p99", passed=..., detail="1.85 <= 2.00")

# Pulls metrics.per_pod[pod][metric][aggregate] for every pod, combines with
# reduce(values), threshold-checks the result. Default reducer is max
# ("did any pod hit the bound?"). Pass any callable that consumes an iterable
# of floats: max, min, sum, statistics.mean, or a lambda around functools.reduce.
metrics.check_per_pod("vllm:kv_offload_store_bytes", "max", ">", 0.0)
metrics.check_per_pod("vllm:kv_cache_usage_perc", "mean", "<=", 80.0, reduce=statistics.mean)
# → Check(name="vllm:foo.max (per-pod max)", passed=..., detail="1.02e+09 > 0")
```

### Custom (non-metric) checks

Construct a `v.Check` directly:

```python
v.Check("PVC has KV cache data", passed=n > 0, detail=f"{n} files at /mnt/kv-cache")
```

### Finish

```python
# Prints the standard report + returns True/False so main() can pick the exit code.
passed = v.print_metrics_verification(env, metrics, checks)
sys.exit(0 if passed else 1)
```

### Aggregates and ops

`process_metrics.py`'s `_compute_stats` produces:
`mean, stddev, min, p25, p50, p75, p90, p95, p99, max, count`

Ops: `<=, >=, <, >, ==`

## Workflow env contract

| Variable | Meaning |
|---|---|
| `LLMDBENCH_WORKSPACE` | Runner workspace; results dir lives below here |
| `LLMDBENCH_CICD_NS` | Namespace used for the run (use for `kubectl -n`) |
| `LLMDBENCH_CICD_SCENARIO` | Scenario name (e.g. `tiered-prefix-cache`) |
| `LLMDBENCH_CICD_WORKLOAD` | Workload filename (e.g. `tiered-prefix-cache.yaml`) |
| `LLMDBENCH_CICD_HARNESS` | Harness name (e.g. `inference-perf`) |
| `LLMDBENCH_CICD_DETECTED_MODEL` | Model id |
| `LLMDBENCH_CICD_OFFLOADING_TARGET` | `'fs'` / `'cpu'` (tiered-prefix-cache mode) |
| `GITHUB_STEP_SUMMARY` | Markdown file to append a report to |
| `GITHUB_RUN_ID` | For traceability |
| `KUBECONFIG` / `~/.kube/config` | kubectl is already configured |

Exit code: `0` on success, non-zero on any failure. Read the CI logs to see
why.

## Required precondition: monitoring on

Any script that reads `metrics_summary.json` requires the harness to scrape
vLLM `/metrics` during the run (`metricsScrapeEnabled: true`). The reusable
workflow forces this on whenever `verify_script` is set — see the
`prereqs_setup_extrac_cli_parms_llmdbenchmark` step. If a caller sets both
`verify_script` and `monitoring_enabled: false`, the workflow fails fast
at the prereqs stage.
