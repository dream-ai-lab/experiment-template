---
name: eval-workflow
description: >-
  The dream-ai-lab evaluation workflow — reproduce a paper, run an eval, and log
  results to Weights & Biases under the shared standard (eval-lib + eval_spec.yaml
  + paper-registry). Use this whenever the user wants to reproduce a paper, run or
  set up an experiment/evaluation, log or push results to W&B / wandb, fill or pin
  an eval_spec, add or use a new metric, propose an improvement over a baseline, or
  register/promote a baseline — even if they don't name "eval-lib" or "the standard"
  explicitly. Prefer this over ad-hoc logging so every run is reproducible and
  comparable.
---

# The dream-ai-lab eval workflow

This repo evaluates a paper under the shared standard. The whole point is that a
number is only worth anything if someone else can **reproduce** it and **compare**
it. The standard buys that with three pieces — don't reinvent them:

- **`eval-lib`** (pip-installed, pinned) — validates the spec, logs the golden
  record to W&B. You never copy or fork it.
- **`eval_spec.yaml`** — the contract: which dataset + model (pinned to exact
  commits), which metrics, what counts as a successful reproduce. Its hash ties a
  result back to the exact config.
- **`paper-registry`** — the catalog of *canonical* specs. A result becomes a
  shared baseline by being registered there.

Every run logs a **golden record** to W&B (provenance + pinned identity), grouped
by `paper_id`, tagged `eval_tier=standard` or `experimental`. Two runs are
comparable iff they share `eval_spec_hash` **and** either `metric_lib_version`
(you used blessed metrics) or `code_fingerprint` (you brought your own).

## Steps

### 1. Get a repo

If you don't already have an experiment repo, create one from the template (it
ships this skill and a working DistilBERT-SST-2 example):

```bash
gh repo create dream-ai-lab/reproduce-<paper-id> \
  --template dream-ai-lab/experiment-template --public --clone
```

### 2. Point W&B at the team (once)

Runs go to Weights & Biases, not a local file. Set this up before running:

```bash
wandb login                          # or export WANDB_API_KEY=...
export WANDB_ENTITY=dream-ai-lab     # the team
export WANDB_PROJECT=eval-lib        # optional; defaults to "eval-lib"
# export WANDB_MODE=offline          # no network now; `wandb sync` later
```

### 3. Fill `eval_spec.yaml`

Pin everything that moves the number. Get real Hugging Face commit SHAs — never
leave `main`/`latest`, the spec is rejected if you do:

```bash
curl -s https://huggingface.co/api/models/<org/model> \
  | python -c "import sys,json;print(json.load(sys.stdin)['sha'])"
```

Pick `metrics.primary` from `eval-lib` (`accuracy`, `f1`, `f1_macro`). **Need a
metric eval-lib doesn't ship yet? Don't wait for a PR** — declare it experimental
and supply it at runtime (step 4b). Set `reproduce_target` to the range that
counts as a successful reproduce.

### 4. Produce results and log them

Pick the path that fits the evaluation. Both log the same golden record.

**4a. Classification (labels) — use `run_paper`.** Write only
`model_fn(texts) -> list[int]`; eval-lib loads the pinned split, runs it, computes
the blessed metrics, and logs:

```python
from eval_lib import load_spec, run_paper
spec = load_spec("eval_spec.yaml")
run_paper("eval_spec.yaml", build_model_fn(spec), role="reproduce")
```

For an experimental metric, declare it in the spec and pass the callable — the run
is tagged `eval_tier=experimental` and fingerprinted, so it can't be mistaken for
a baseline:

```yaml
metrics: { primary: "mcc", experimental: ["mcc"] }
```
```python
def mcc(preds, refs): ...                         # (preds, refs) -> float
run_paper("eval_spec.yaml", model_fn, extra_metrics={"mcc": mcc})
```

**4b. Anything else (generative win-rate, MT-Bench, an LLM judge) — use
`log_run`.** These don't fit `model_fn -> labels`. Compute the numbers with your
own harness, then just record them. Pass `code_fingerprint` (so it's tagged
experimental and pinned to your eval code) and attach judge config / generations:

```python
from eval_lib import log_run
results = {"alpaca_win_rate": 0.62, "mt_bench": 7.8}      # YOUR harness produced these
log_run(
    "eval_spec.yaml", results, role="proposal", parent_run_id=baseline_run_id,
    code_fingerprint=git_sha("eval/"),
    params={"judge": "gpt-4o-2024-08-06", "judge_temp": 0.0},
    artifacts=["eval/judge_prompt.txt", "outputs/generations.jsonl"],
)
```

### 5. Read the result

`run_paper`/`log_run` return `{run_id, results, reproduce_passed}` and the run
appears in W&B under your project, grouped by `paper_id`. `reproduce_passed=True`
means every `reproduce_target` metric landed in range. A `proposal` run with a
`parent_run_id` also logs `delta_<metric>` vs the baseline.

### 6. Register a baseline (only when it stabilises)

Keep iterating in *this* repo as long as you like — nothing to register. **Once a
result will become a baseline others compare to**, open a PR adding its
`eval_spec.yaml` to [`paper-registry`](https://github.com/dream-ai-lab/paper-registry).
Rule of thumb: *register before a result becomes a baseline*, and never let two
divergent specs exist for one `paper_id`. Experimental-metric specs are **not**
accepted there (the registry holds canonical contracts only) — promote first.

### 7. Promote an experimental metric → standard

When an experimental metric stabilises, PR the function into
[`eval-lib`](https://github.com/dream-ai-lab/eval-lib), bump its version, and drop
the name from `metrics.experimental`. The run then logs `eval_tier=standard`
automatically and becomes org-wide comparable.

## What you never do

- Copy or fork `eval-lib` — depend on a **pinned version** (`requirements.txt`).
- Re-implement a *standard* metric, or invent a one-off eval function — use it by
  name, or run it experimental now and promote it later.
- Log results ad-hoc (a stray sheet, a bare `wandb.log`) — go through
  `run_paper`/`log_run` so the golden record and comparability come for free.
- Leave a dataset/model on `main`/`latest` — pin to a commit SHA.
