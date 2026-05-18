# Kaggle Workflow Guide: Fine-tuning Gemma 4 for Screen Grounding

How to actually use Kaggle as your training platform when your code lives in a repo. Kaggle gives you free GPU access (T4×2, P100, or L4 depending on the moment), 30 hours/week of GPU time on the free tier, up to 12 hours per session. Enough for this project if you structure things right.

## The mental model

Kaggle has three primitives you'll use:

- **Datasets** — arbitrary file uploads, mounted read-only at `/kaggle/input/<slug>/` during runtime. Up to 100GB each. Versioned.
- **Models** — pretrained model weights, also mounted at `/kaggle/input/`. Often faster than downloading from HF.
- **Notebooks (Kernels)** — the actual compute. Pick a GPU, pick inputs, run code. Outputs to `/kaggle/working/`.

Your code repo becomes a **Dataset**. Your training data becomes another **Dataset**. The base model goes through Kaggle's **Models** interface. Your notebook ties them together.

Outputs (checkpoints, logs, eval results) live in `/kaggle/working/` during the session and become part of the notebook's saved output, or you push them to a new Dataset version.

## Step-by-step workflow

### Step 1: Prepare the code repo

Before uploading, make sure your repo is *Kaggle-ready*:

- It must run from a single entry point (e.g., `python -m src.train.train --config configs/exp002.yaml`).
- All paths in configs use environment variables or relative paths, never `/home/yourname/...`.
- `requirements.txt` is complete and pinned.
- Add a `kaggle/` directory with helper scripts (see below).

Add this to your repo:

```
kaggle/
├── notebook_template.py        # The code your Kaggle notebook will run
├── install_deps.sh             # Pip install commands for the Kaggle env
└── README_KAGGLE.md            # Reminder of how to use this on Kaggle
```

### Step 2: Upload code as a Kaggle Dataset

You have two options:

**Option A: GitHub-as-dataset (cleanest).** Kaggle supports pulling a public GitHub repo directly. In Datasets → New Dataset → "Link from GitHub", paste your repo URL. Every time you push to GitHub, you re-version the Kaggle Dataset with one click.

**Option B: Manual upload.** Zip your repo (excluding `data/`, `checkpoints/`, etc.) and upload. Slower to iterate but works for private repos.

Either way, your code ends up at something like `/kaggle/input/gemma4-screen-grounding/` inside the notebook.

**Naming convention.** Give the dataset a stable slug, e.g., `your-username/gemma4-grounding-code`. You'll reference it in many notebooks.

### Step 3: Upload data as a separate Dataset

Even though Kaggle Datasets can hold 100GB, you want this small (~1-2GB) for fast notebook startup. Strip OS-Atlas down to just what you need:

```
gemma4-grounding-data/
├── images/                      # Resized JPEGs, ~500MB
├── annotations/
│   ├── train.json               # Your unified, filtered, formatted train set
│   └── val.json
└── screenspot_v2/               # The eval set
    ├── images/
    └── annotations.json
```

Do the resizing and filtering on your local machine *before* uploading. Don't make Kaggle do this work every session.

Upload as `your-username/gemma4-grounding-data`. Version it when you change the mix.

### Step 4: Add the base model via Kaggle Models

Kaggle has Gemma models available through the Models tab. Search for `unsloth/gemma-4-E4B-it` or similar. Add it to your notebook as an input — this avoids a slow HF download every session.

If the exact Unsloth variant isn't there, you can pin Google's Gemma 4 E4B from Kaggle Models and let Unsloth's `FastVisionModel.from_pretrained()` load from the local path.

### Step 5: Create the training notebook

A minimal Kaggle training notebook (`gemma4-train`) looks like this:

```python
# Kaggle notebook: gemma4-train

# ----- Setup -----
import os, sys, subprocess

# Make code repo importable
CODE_PATH = "/kaggle/input/gemma4-grounding-code"
sys.path.insert(0, CODE_PATH)

# Install pinned deps
subprocess.run(["bash", f"{CODE_PATH}/kaggle/install_deps.sh"], check=True)

# Set tokens via Kaggle Secrets (Add-ons > Secrets in notebook UI)
from kaggle_secrets import UserSecretsClient
secrets = UserSecretsClient()
os.environ["HF_TOKEN"] = secrets.get_secret("HF_TOKEN")
os.environ["WANDB_API_KEY"] = secrets.get_secret("WANDB_API_KEY")

# ----- Paths -----
DATA_PATH = "/kaggle/input/gemma4-grounding-data"
MODEL_PATH = "/kaggle/input/gemma-4/transformers/e4b-it/1"  # adjust to actual model slug
OUTPUT_DIR = "/kaggle/working/checkpoints"

# ----- Run training -----
from src.train.train import run_training

run_training(
    config_path=f"{CODE_PATH}/configs/exp002_lora_r32.yaml",
    data_dir=DATA_PATH,
    model_path=MODEL_PATH,
    output_dir=OUTPUT_DIR,
)
```

The key idea: **the notebook is thin glue**. All real logic lives in your repo. The notebook handles env setup, secret loading, and path wiring.

### Step 6: Notebook settings

In the notebook editor's right sidebar:

- **Accelerator:** GPU → pick best available (P100 16GB or L4 24GB beats T4×2 for VLM training because you can't easily split a single Unsloth run across two T4s).
- **Internet:** ON. Needed for installing packages and W&B logging. Off by default; you must verify your account (phone) to enable.
- **Persistence:** "Files only" is fine. Checkpoints in `/kaggle/working/` will persist as notebook output.
- **Environment:** Pin to a specific version (the "Pin to environment" toggle) so your run is reproducible.

## Working around the 12-hour session limit

A full training run might exceed 12 hours. Strategy: **train in chunks, save checkpoints to a Dataset, resume next session**.

Pattern:

1. Run training session 1, save the final checkpoint to `/kaggle/working/`. When the session ends, the output becomes downloadable.
2. Download the checkpoint locally, re-upload it as a Kaggle Dataset (or add a version to an existing one): `your-username/gemma4-grounding-checkpoints`.
3. In session 2, add the checkpoint dataset as input. Your training script reads the checkpoint and resumes.

To avoid the download/re-upload friction, you can also use the **Kaggle API** to push outputs as a new dataset version directly:

```python
# At end of training session
import subprocess
subprocess.run([
    "kaggle", "datasets", "version",
    "-p", OUTPUT_DIR,
    "-m", "Checkpoint after step 4000",
    "-r", "zip",
], check=True)
```

This requires `kaggle.json` API credentials in `/root/.kaggle/` — also loaded via Kaggle Secrets.

## GPU quota management

Free tier is 30 hours/week of GPU time. Treat this as the most precious resource. Practical tips:

- **Don't iterate training code on a GPU.** Develop on CPU/no-accelerator sessions, only switch to GPU for actual training runs.
- **Smoke-test small.** Before a 6-hour run, do a 100-step run to verify everything works. ~5 minutes of GPU time.
- **Monitor in W&B.** Watch the run remotely; you can kill it from the Kaggle notebook UI if loss diverges, saving hours.
- **Stop sessions you don't need.** When training finishes, manually stop the kernel — Kaggle will sometimes leave it running.

If 30 hours becomes a real constraint, options:
- Kaggle's paid "Notebooks Pro" tier
- Move to Colab Pro+ (similar money, similar restrictions, sometimes better GPUs)
- Lambda Labs / Vast.ai for ad-hoc rented GPUs (cheap, no quota, but you manage the machine)

## Common Kaggle gotchas

**Internet off by default.** You must enable it for every notebook AND verify your phone number on your account first. Pip installs will silently fail if internet is off — error messages don't always make this obvious.

**`/kaggle/working/` has a 20GB limit.** Big checkpoints + W&B local logs can hit this. Periodically clear stale stuff during training:
```python
import shutil
shutil.rmtree("/kaggle/working/old_checkpoint", ignore_errors=True)
```

**Notebook outputs are zipped and can be slow to commit.** A "Save Version" with 5GB of output can take 10+ minutes after training finishes. Plan for it.

**Secrets must be added per-notebook, not globally.** First time you use a notebook, go to Add-ons → Secrets and attach each one.

**Default Python environment is huge but missing things.** Kaggle preinstalls a lot, but Unsloth specifically needs careful version handling. Your `install_deps.sh` should be:
```bash
#!/bin/bash
pip install --upgrade -q "unsloth[colab-new] @ git+https://github.com/unslothai/unsloth.git"
pip install --no-deps -q "xformers<0.0.27" trl peft accelerate bitsandbytes
pip install -q wandb omegaconf
```
This matches Unsloth's Colab pattern, which is the most-tested install path.

**Long-running sessions get killed for idle.** If your training script takes 10 hours and doesn't produce output, Kaggle might kill it for inactivity. Always log to W&B at least every few minutes; this counts as activity.

**You can't SSH or get a shell.** Everything happens through the notebook UI or the Kaggle API. Bash works inside cells via `!command` or `%%bash`.

## A complete first-day workflow

Concrete plan for your first Kaggle training session:

1. **Local prep (1 hour):**
   - Resize and subsample OS-Atlas → ~10K samples, ~1GB
   - Format to Gemma 4 chat format → `train.json`
   - Package as `gemma4-grounding-data/` directory
   - Upload to Kaggle as a new Dataset

2. **Code dataset (15 min):**
   - Push current repo to GitHub
   - Link from Kaggle as a Dataset
   - Or zip-upload manually

3. **Model (5 min):**
   - Add Gemma 4 E4B from Kaggle Models

4. **Notebook (30 min):**
   - Create new notebook, attach all 3 inputs
   - Add HF_TOKEN and WANDB_API_KEY secrets
   - Paste the boilerplate from Step 5 above
   - Run a 50-step smoke test

5. **Real training (4-8 hours):**
   - Verify the smoke test produced sensible loss
   - Switch to full run with your real config
   - Monitor via W&B on your phone
   - Final checkpoint saves to `/kaggle/working/`

6. **Wrap-up (30 min):**
   - Click "Save Version" to persist outputs
   - Download the checkpoint or push it as a new Dataset version
   - Note the W&B run URL in your experiment tracker

That's an end-to-end workflow within one Kaggle session. From here, iteration just means: tweak config → new notebook version → new training run.

## Things you should pin once and forget

- The dataset slug for code, data, and checkpoints
- The Kaggle Models slug for the base model
- Your W&B project name and entity
- The `install_deps.sh` versions
- Your `.kaggle/kaggle.json` API credentials (locally)

Once these are stable, starting a new experiment is "duplicate notebook, change config file path, run."