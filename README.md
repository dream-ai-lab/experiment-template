# experiment-template

GitHub **template repo** for reproducing a new paper under the eval standard.
Ships a working DistilBERT-SST-2 example so it runs green immediately — you
replace it with your paper.

> **Working in Claude Code?** This repo ships the **`eval-workflow`** skill
> (`.claude/skills/`). Just say what you want ("reproduce paper X", "log this run",
> "add a metric") and it walks you through the whole flow end-to-end.

## When you want to experiment with a new paper

1. Click **“Use this template” → Create a new repository**, named
   `reproduce-<paper-id>`, in the `dream-ai-lab` org.
   (CLI: `gh repo create dream-ai-lab/reproduce-<paper-id> --template dream-ai-lab/experiment-template --public --clone`)

2. **Fill `eval_spec.yaml`** (survey member) — pin dataset + model to real HF
   commit SHAs, pick `metrics.primary` from `eval-lib` (`accuracy`, `f1`,
   `f1_macro`), set `reproduce_target`. Get SHAs:
   ```bash
   curl -s https://huggingface.co/api/models/<org/model> | python -c "import sys,json;print(json.load(sys.stdin)['sha'])"
   ```
   Need a metric `eval-lib` doesn't ship yet? Don't wait for a PR — declare it
   experimental and run now (see *Experimental metrics* below).

3. **Edit `model_fn`** in `reproduce.py` (experiment member) — map the model's
   output labels to the dataset's label ids; for non-classification tasks,
   replace the body. The contract is just `model_fn(texts) -> list[int]`.

4. **Point W&B at the team** (runs log to Weights & Biases):
   ```bash
   wandb login                          # or set WANDB_API_KEY
   export WANDB_ENTITY=dream-ai-lab     # the team
   export WANDB_PROJECT=eval-lib        # optional; defaults to "eval-lib"
   # export WANDB_MODE=offline          # run now without a network; `wandb sync` later
   ```

5. **Run:**
   ```bash
   pip install -r requirements.txt        # pulls eval-lib (pinned) + wandb + torch + transformers
   python reproduce.py
   ```
   If `target_passed=True`, you reproduced it. The run lands in W&B under your
   project, grouped by `paper_id`.

6. **Propose** (optional): add `proposal.py`, log with `role="proposal"` and a
   `parent_run_id` to get `delta_<metric>` vs the baseline — still no PR to any
   central repo.

## Experimental metrics — a metric `eval-lib` doesn't have yet

You do **not** need to PR a metric into `eval-lib` before you can experiment.
Declare it in the spec and pass the callable at runtime:

```yaml
# eval_spec.yaml
metrics:
  primary: "mcc"
  experimental: ["mcc"]     # not in eval-lib; supplied below
```

```python
# reproduce.py
def mcc(preds, refs):       # your metric, (preds, refs) -> float
    from sklearn.metrics import matthews_corrcoef
    return float(matthews_corrcoef(refs, preds))

run_paper(SPEC, build_model_fn(spec), extra_metrics={"mcc": mcc})
```

The run is tagged `eval_tier=experimental` (vs `standard`) and records a
fingerprint of your metric code, so it stays clearly separate from official
baselines. **Promote** it once it stabilises: PR the function into `eval-lib`,
bump the version, and drop it from `metrics.experimental` — the run then logs
`eval_tier=standard` automatically. Requires `eval-lib >= 0.2.0`.

## What you never do

- Copy or fork `eval-lib` — depend on a **pinned version** (`requirements.txt`).
- Re-implement a *standard* metric — use it by name, or (for a new one) run it
  experimental now and promote it into `eval-lib` later.
- PR experiment code back to a central repo — this repo is yours.

No central PR is needed to run an experimental metric. The only central PRs are:
promoting a metric → `eval-lib`, a new canonical spec → `paper-registry`.
