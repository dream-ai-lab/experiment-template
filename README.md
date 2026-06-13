# experiment-template

GitHub **template repo** for reproducing a new paper under the eval standard.
Ships a working DistilBERT-SST-2 example so it runs green immediately — you
replace it with your paper.

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

3. **Edit `model_fn`** in `reproduce.py` (experiment member) — map the model's
   output labels to the dataset's label ids; for non-classification tasks,
   replace the body. The contract is just `model_fn(texts) -> list[int]`.

4. **Run:**
   ```bash
   pip install -r requirements.txt        # pulls eval-lib (pinned) + torch + transformers
   python reproduce.py
   ```
   If `target_passed=True`, you reproduced it. Log to the shared server with
   `MLFLOW_TRACKING_URI=http://<server>:5000`.

5. **Propose** (optional): add `proposal.py`, fork the baseline run — still no
   PR to any central repo.

## What you never do

- Copy or fork `eval-lib` — depend on a **pinned version** (`requirements.txt`).
- Write your own metric — add it to `eval-lib` and bump its version.
- PR experiment code back to a central repo — this repo is yours.

The only central PRs are: a new metric → `eval-lib`, a new spec → `paper-registry`.
