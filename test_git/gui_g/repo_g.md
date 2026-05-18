# Repo Structure Guide: Gemma 4 Screen Grounding Project

A working layout for an ML portfolio project that does VLM fine-tuning, on-device deployment, and includes a demo app. Designed so each piece is independently testable and the whole thing reads as professional ML engineering work.

## Top-level layout

```
gemma4-screen-grounding/
├── README.md                    # Portfolio-facing pitch + quickstart
├── pyproject.toml               # Or requirements.txt — pin versions
├── .gitignore
├── .env.example                 # Template for HF_TOKEN, WANDB_API_KEY, etc.
│
├── configs/                     # YAML configs — one per experiment
│   ├── base.yaml
│   ├── exp001_baseline_eval.yaml
│   ├── exp002_lora_r32.yaml
│   └── exp003_lora_r64_unfrozen_vision.yaml
│
├── src/
│   ├── data/                    # Data pipeline
│   │   ├── download.py          # Selective HF snapshot download
│   │   ├── adapters/            # One file per source
│   │   │   ├── __init__.py
│   │   │   ├── amex.py
│   │   │   ├── seeclick.py
│   │   │   ├── uibert.py
│   │   │   └── widget_caption.py
│   │   ├── filter.py            # Per-sample sanity checks
│   │   ├── format_gemma4.py     # Convert to Gemma 4 chat format
│   │   ├── mix.py               # Multi-source weighted sampling
│   │   └── resize.py            # One-time image resize pass
│   │
│   ├── eval/                    # Evaluation harness — independent of training
│   │   ├── screenspot_v2.py     # Main eval driver
│   │   ├── metrics.py           # Point-in-box, IoU, stratified accuracy
│   │   ├── prompts.py           # Prompt variants for baselining
│   │   └── visualize.py         # Failure overlay plots
│   │
│   ├── train/                   # Training code
│   │   ├── train.py             # Main entry point — reads config, runs Unsloth
│   │   ├── callbacks.py         # W&B logging, periodic ScreenSpot eval
│   │   └── collator.py          # Wraps UnslothVisionDataCollator if needed
│   │
│   └── deploy/                  # Post-training deployment
│       ├── quantize.py          # INT8/INT4 quantization
│       ├── export_litert.py     # Conversion to .litertlm format
│       ├── export_gguf.py       # Unsloth → GGUF for llama.cpp
│       └── benchmark_device.py  # On-device latency/memory profiling
│
├── android/                     # Demo Android app (separate Gradle project)
│   ├── app/
│   ├── build.gradle
│   └── README.md
│
├── scripts/                     # Thin orchestration shell scripts
│   ├── 00_setup.sh
│   ├── 01_download_data.sh
│   ├── 02_baseline_eval.sh
│   ├── 03_train.sh
│   └── 04_eval_checkpoint.sh
│
├── notebooks/                   # For ANALYSIS, not training
│   ├── 01_data_inspection.ipynb       # Visual spot-check of samples
│   ├── 02_baseline_prompts.ipynb      # Zero-shot prompt comparison
│   ├── 03_failure_analysis.ipynb      # Stratified breakdown of errors
│   └── 04_quantization_impact.ipynb   # FP16 vs INT8 vs INT4
│
├── tests/                       # Unit tests
│   ├── test_adapters.py
│   ├── test_format_gemma4.py
│   ├── test_metrics.py
│   └── test_eval_harness.py
│
└── docs/
    ├── design.md                # Project design decisions
    ├── results.md               # Numbers, ablations, leaderboard table
    └── blog_post_draft.md       # Eventual public writeup
```

## Why this structure

**`src/` is split by concern, not by file type.** Data, eval, train, deploy each form a coherent module that can be reasoned about independently. Eval especially should NEVER import from train — it should work on any HuggingFace-format checkpoint, including pretrained ones you didn't train.

**`configs/` holds one YAML per experiment.** Never hard-code hyperparameters in scripts. Every training run is `python -m src.train.train --config configs/exp002_lora_r32.yaml`. This makes every result reproducible from a single file you can commit to git.

**`notebooks/` is for analysis only, not training.** Notebooks are great for inspecting data and visualizing failures, terrible for running long training jobs. Training goes in `src/train/train.py` driven by configs. Notebooks read checkpoints and configs as inputs.

**`android/` is a sibling project, not a submodule.** It has its own Gradle build. Keeping it in the same repo is fine for portfolio purposes but understand it's a separate buildable unit.

**`scripts/` are thin wrappers.** Each shell script is 5-20 lines that calls the Python modules with the right config and arguments. The logic lives in Python, not in bash. The point of `scripts/` is reproducibility: anyone can run `bash scripts/02_baseline_eval.sh` to reproduce your baseline.

## What goes in `configs/base.yaml`

```yaml
# configs/base.yaml — shared defaults
project: gemma4-screen-grounding
seed: 42

model:
  name: unsloth/gemma-4-E4B-it
  max_seq_length: 4096
  load_in_4bit: true
  full_finetuning: false

data:
  root_dir: ./data
  sources:
    amex:
      weight: 0.40
      max_samples: 10000
    seeclick_web:
      weight: 0.20
      max_samples: 5000
    # ...
  image_max_long_side: 1120
  visual_token_budget: 560

train:
  output_dir: ./checkpoints
  batch_size: 1
  grad_accum_steps: 4
  learning_rate: 2e-4
  max_steps: 2000
  warmup_steps: 50
  save_every: 500
  eval_every: 500

lora:
  r: 32
  alpha: 32
  dropout: 0
  finetune_vision_layers: false
  finetune_language_layers: true

eval:
  screenspot_path: ./data/screenspot_v2
  generation:
    temperature: 0.0
    max_new_tokens: 64

wandb:
  project: gemma4-grounding
  entity: your-wandb-username
```

Experiment configs extend `base.yaml` with overrides. Use a config loader (Hydra, OmegaConf, or a 50-line custom one) that merges them.

## Things that look small but matter a lot

**`.env.example`.** Commit a template showing what env vars are needed (HF_TOKEN, WANDB_API_KEY). Real `.env` is gitignored. Reviewers can clone your repo and set up env vars in 60 seconds.

**`requirements.txt` with pinned versions.** Every package gets a `==`. `transformers==4.55.0`, not `transformers>=4.55`. VLM training is famously fragile to version skew.

**A `make baseline`, `make train`, `make eval` Makefile (optional but slick).** Lets reviewers run common operations without reading any docs.

**Per-checkpoint metadata.** When `train.py` saves a checkpoint, also save the config used and the git commit hash in the same directory:
```
checkpoints/exp002_lora_r32/
├── adapter_model.safetensors
├── config.yaml                   # Copy of the run config
├── train_metadata.json           # {git_sha, timestamp, dataset_hash, ...}
└── val_metrics.json
```
This means six months later you can reproduce *exactly* what a given checkpoint was trained on.

## README content (the most-read file)

Your README is what hiring managers actually look at. Structure it as:

1. **One-line pitch.** "On-device GUI grounding using fine-tuned Gemma 4 E4B, hitting X% on ScreenSpot-V2 on Pixel 7 at <1s/inference."
2. **Headline number/result.** Big, bold, with a comparison to a known baseline.
3. **Demo GIF.** A 15-second screen recording of the Android app working. This is the most important single asset.
4. **Architecture diagram.** One image. Boxes for: data sources, training pipeline, exported model, Android app.
5. **Reproducibility section.** "To reproduce the baseline: `bash scripts/02_baseline_eval.sh`. To train: ..."
6. **Results table.** ScreenSpot-V2 by platform, FP16 vs INT8 vs INT4, latency on device.
7. **Tech stack callout.** Unsloth, ExecuTorch/LiteRT-LM, W&B, etc.
8. **Links.** Blog post, Android APK release, HuggingFace model card.

Do NOT structure your README as a step-by-step tutorial. Reviewers don't have time. Lead with what you built and the number, then provide a reproducibility section near the bottom for the rare reader who wants to run it.

## What to gitignore

```
# .gitignore
__pycache__/
*.pyc
.env
.venv/
venv/

# Data — never commit
data/raw/
data/processed/
data/screenspot_v2/

# Checkpoints — never commit (use HF or W&B artifacts)
checkpoints/
outputs/
wandb/

# Notebook output
.ipynb_checkpoints/
notebooks/*.ipynb_checkpoints

# OS junk
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
```

Push trained model artifacts to HuggingFace Hub (`hcompany/...`-style), not git. Push experiment artifacts (loss curves, eval results) to W&B. Git is for code only.

## Day-one checklist

When you start the repo, do these in order:

1. `git init`, `.gitignore` from above, basic README with a placeholder.
2. `pyproject.toml` with pinned deps.
3. `configs/base.yaml` skeleton.
4. `src/eval/screenspot_v2.py` — write this BEFORE any training code. It's the most important script in the repo.
5. `tests/test_eval_harness.py` — verify your eval against Holo1.5-3B's published 91.66%.
6. Only then start on `src/data/` and `src/train/`.

This ordering enforces "baseline before training," which is the single discipline most likely to save you from a bad portfolio result.