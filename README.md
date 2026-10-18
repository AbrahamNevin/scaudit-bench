# SCAudit-Bench

A validation harness for the paper **"LoRA-Based Fine-Tuning of Large Language Models for Detecting
Non-Machine-Auditable Smart Contract Vulnerabilities"**. The paper is a research proposal with no results; this
project is the experiment code that produces them. It builds the dataset, runs every baseline, trains LoRA /
full fine-tuning / soft-prompt models, runs the ablations and the Section VII failure-case tests, and writes a
report with a verdict (SUPPORTED / NOT SUPPORTED / INCONCLUSIVE) for each claim.

> **Read this first.** Out of the box the project runs in **smoke mode**: a tiny language model built from
> scratch on your CPU, no downloads, no GPU. Smoke mode proves that the *pipeline, statistics and tests work*; its
> numbers say nothing about Llama3-8B or Qwen2-7B. To test the paper's actual claims, run `configs/full.yaml` on
> a GPU (see "Full run"). Also note that the only price-manipulation positives come from template-generated
> contracts (see `data/DATA_SOURCES.md`), so even full-mode numbers are optimistic until you add real audit data.

## Quick start (VS Code or Antigravity)

```bash
python -m venv .venv
# Windows:  .venv\Scripts\activate        macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt

python -m pytest -m "not slow"                  # ~15 s, unit tests
python run_all.py --config configs/ci.yaml      # ~3 min, tiny end-to-end sanity run
python run_all.py --config configs/smoke.yaml   # full pipeline with the offline tiny model (tens of minutes on a laptop CPU)
```

Then open `results/smoke/report.md` and press `Ctrl+Shift+V` for the Markdown preview. Figures are in
`results/smoke/figures/`, every prediction is in `results/smoke/results.json`. The `.vscode/` folder has ready-made
launch configurations (Run and Debug panel) and test settings. Runs are cached in `results/<name>/runs/`, so if you
stop the process it resumes where it left off (delete the folder to start fresh).

One-liner: `scripts/run_smoke.sh` (macOS/Linux) or `scripts\run_smoke.bat` (Windows) runs the tests and then the smoke experiment.

`example_output/smoke/` holds the report, figures and log from a complete smoke run made while building this project (124 min on a 2-core CPU), so you can see what the output looks like before running anything.

## Full run (tests the paper's claims)

1. A CUDA GPU (16-24 GB for 4-bit LoRA on an 8B model; full fine-tuning of 8B needs far more; see `configs/full.yaml`).
2. `pip install bitsandbytes` (Linux / WSL) for QLoRA, and `huggingface-cli login` (Llama-3 is gated: accept its licence on huggingface.co).
3. Edit `model.hf_name` in `configs/full.yaml` (Llama3-8B, Qwen2-7B or CodeLlama-7B) and run
   `python run_all.py --config configs/full.yaml`. Narrow it with `--set experiments.ablations=[method,augmentation] --set seeds=[0]`.
4. Optional extras that improve fidelity: real Slither/Mythril (below), and your own audit-derived contracts in `data/external/`.

The same code path loads the tiny model and the real checkpoints, so what the tests and smoke run exercise is what the full run uses.

## What the paper promises, and where it is tested

| Paper | What is measured | Code |
|---|---|---|
| V.A data, Fig. 2-3 | SmartBugs Curated + synthetic + optional audit data; label mapping; **project-level** 70/15/15 split with leakage audit; random oversampling and synthetic augmentation | `scaudit/data/` |
| V.B representation, prompts | plain / function-annotated / token-stream inputs; simple, CoT, retrieval, hybrid prompts; open-ended output | `data/representation.py`, `models/prompts.py` |
| V.C-D LoRA, training | PEFT LoRA (rank/alpha/dropout sweep, LR grid), FFT, soft-prompt tuning; early stopping, clipping, mixed precision, QLoRA | `models/trainer.py` |
| VI.A baselines | Slither and Mythril (real, if installed; otherwise offline pattern-rule proxies), zero/few-shot prompting, hybrid static+LLM, soft-prompt (CGP stand-in), majority class | `experiments/core.py`, `models/` |
| VI.B metrics | per-class and macro P/R/F1, ROC/PR curves and AUC, FP/FN rates, Brier, ECE, reliability diagram, paired t-test + McNemar + paired bootstrap, comparison with Bu et al. (0.97 / 0.68) | `eval/`, `experiments/report.py` |
| VI.C ablations | method, augmentation, prompt strategy, model size, feature drop, taxonomy granularity | `experiments/ablations.py` |
| VII failure cases | label noise, duplicate leakage, context loss, over-sensitivity, hallucinated explanations, wrong class, call-order logic flaws | `experiments/failures.py` |
| (verdicts) | one row per claim, with the evidence and a fixed decision rule | `experiments/hypotheses.py` |

### How verdicts work

Thresholds are declared in the config (`eval.*`) before running. **SUPPORTED** = the effect is at least the declared
size and its 95% bootstrap interval (or an exact test) excludes zero; **NOT SUPPORTED** = the interval's upper bound is below the declared
size; **INCONCLUSIVE** = in between, typically because the test set has few positives for a rare class. A sanity
gate (H0) withholds all other verdicts if the LoRA model fails to beat a trivial "flag everything" classifier. Negative
results are reported as such; they are valid outcomes for a proposal.

## Real Slither / Mythril

```bash
pip install slither-analyzer solc-select        # mythril optional: pip install mythril
python scripts/setup_static_tools.py            # installs the solc versions the dataset's pragmas need (internet)
```

If a tool or a matching compiler is missing, it is reported as *skipped*, never silently replaced. Contracts a tool
cannot compile count as "no finding", and the analysed fraction is printed next to its score. The parsers for the tools'
JSON output are unit-tested on recorded fixtures only: they were written without the real binaries available, so if
your Slither/Mythril version formats output differently, `scaudit/models/static_tools.py` is the place to adjust.
The "Pattern rules" rows are *proxies* (reentrancy pattern, optional spot-price rule), not the real tools.

## Adding real data

Put JSON Lines in `data/external/custom_audits.jsonl` (format in `data/external/README.md`). Records are grouped by
`project` for the split, and the number confirmed by two or more auditors is reported (the paper's labelling rule).
Set `data.synthetic.n_projects: 0` to drop the synthetic contracts entirely.

## Layout

```
run_all.py            entry point (see --help)
configs/              smoke.yaml (offline CPU) | full.yaml (GPU, real LLMs) | ci.yaml (tiny, used by tests)
scaudit/data/         loaders, synthetic generator, split + leakage audit, augmentation, representations
scaudit/models/       prompts, LLM loading/scoring, LoRA/FFT/prompt trainer, rules, Slither/Mythril wrappers, tiny model
scaudit/eval/         metrics, calibration, statistical tests
scaudit/experiments/  baselines, ablations, failure tests, hypotheses, report
tests/                pytest suite (36 fast tests + 1 end-to-end test)
data/raw/             SmartBugs Curated (bundled)      data/external/   your own audit data
results/<config>/     report.md, results.json, figures/, runs/ (cached), data/ (the splits as JSONL), run.log
```

## Known limitations (also stated in every report)

- Smoke mode uses a ~0.6M-parameter model; its results validate the harness, not the paper.
- The label mapping from SmartBugs' DASP categories to the paper's three classes is an assumption (editable in the config).
- Synthetic contracts have exact labels but share templates; real-world difficulty is higher.
- The "AST-tokenized" representation is a lexical token stream (a true AST needs a working solc).
- Soft-prompt tuning is a simplified stand-in for CGP-Tuning (no graph encoder); GPT-3.5/4 few-shot reference arms are not included (they need an API key).
- Ablation arms use one seed; only the LoRA / FFT / soft-prompt comparison uses all `seeds`.
- DiverseVul and the Etherscan / audit-report corpora are not bundled (see `data/DATA_SOURCES.md`).
# scaudit-bench
