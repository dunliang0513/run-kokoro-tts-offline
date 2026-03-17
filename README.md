# How to run Kokoro TTS offline without triggering the Network Error

Patches for running Kokoro TTS fully offline by loading model weights,
config, and voice files from a local directory instead of downloading from HuggingFace Hub on every run.

## Problem

By default, the Kokoro TTS library unconditionally calls `hf_hub_download()` at runtime — even if the
model files are already present locally. This causes inference to fail in offline or air-gapped environments
(e.g. Nvidia Jetson devices without internet access).

The affected source files are:
- `kokoro/model.py` — downloads `config.json` and `.pth` model weights from HuggingFace on every init
- `kokoro/pipeline.py` — downloads voice `.pt` files from HuggingFace on every voice load

## Solution

Patch both source files inside your virtual environment to prioritize a local directory,
with HuggingFace download demoted to a fallback only when a file is genuinely missing.

### Key Changes

| File | What was changed |
|---|---|
| `model.py` | Added `local_model_dir` param to `KModel.__init__`; loads `config.json` and `.pth` from local path |
| `pipeline.py` | Added `local_voices_dir` param to `KPipeline.__init__`; loads voice `.pt` files from local path |

`hf_hub_download` is now a lazy import inside fallback blocks only, so the network is never touched
if all files are present locally.

## Local Directory Structure

Organize your downloaded files as follows before applying the patch:

```
models/tts_kokoro/
├── config.json
├── kokoro-v1_0.pth
└── voices/
    ├── af_heart.pt
    ├── af_bella.pt
    ├── am_michael.pt
    └── <any other voices>.pt
```

## How to Apply

### 1. Download model and voice files from HuggingFace (one-time, while online)

From the Kokoro HuggingFace page:

- Download `kokoro-v1_0.pth` and `config.json` — place them in your local model directory
- Download any voice `.pt` files you need from the `voices/` folder — place them inside a `voices/` subfolder in the same directory

### 2. Locate the Kokoro source files in your venv

```bash
find .venv -path "*/kokoro/model.py"
find .venv -path "*/kokoro/pipeline.py"
```

### 3. Apply Patched Files

Copy the patched `model.py` and `pipeline.py` into your virtual environment:

```bash
cp model.py pipeline.py .venv/lib/python3.10/site-packages/kokoro/
```

**Note:** The exact path may vary. Find it with:
```bash
python -c "import kokoro; print(kokoro.__file__)"
cd $(dirname $(dirname $(python -c "import kokoro; print(kokoro.__file__)")))
```

### 4. Configure Local Model Paths

The patched files use **automatic path detection** that walks up from the installed package location to find your project root containing `models/tts_kokoro/`.

**Primary method (automatic):**
```python
# Already included in patched files - no action needed!
# Walks up until it finds models/tts_kokoro/voices → /home/och/Demo/models/tts_kokoro
```

**Manual override (backup):**
If automatic detection fails (e.g., unusual project layout), set the hardcoded fallback:

In `model.py`:
```python
# Fallback path - change to your actual project root
PROJECT_ROOT = pathlib.Path("/home/och/Demo")  # ← Update this line only if needed

DEFAULT_LOCAL_MODEL_DIR = PROJECT_ROOT / "models" / "tts_kokoro"
```

In `pipeline.py`:
```python
PROJECT_ROOT = pathlib.Path("/home/och/Demo")  # ← Update this line only if needed

DEFAULT_LOCAL_VOICES_DIR = PROJECT_ROOT / "models" / "tts_kokoro" / "voices"
```

### 5. Verify the Paths

Test the resolution:
```bash
source .venv/bin/activate
python -c "
import kokoro.model as m; print('Model dir:', m.DEFAULT_LOCAL_MODEL_DIR)
import kokoro.pipeline as p; print('Voices dir:', p.DEFAULT_LOCAL_VOICES_DIR)
"
```

Expected output:
```
/home/och/Demo/models/tts_kokoro
/home/och/Demo/models/tts_kokoro/voices
```

The manual paths are only used as a **fallback** if the automatic detection can't find the `models/tts_kokoro` directory anywhere in the parent chain.

## Notes

- These patches were tested on **Kokoro 82M v1.0** (`hexgrad/Kokoro-82M`) on an **Nvidia Jetson Orin Nano**
  running **Python 3.10**
- If a voice file is missing locally, a warning will be logged and it will fall back to downloading from HuggingFace
- If you upgrade the Kokoro package, re-apply the patches as the source files will be overwritten
- To avoid re-patching on every upgrade, consider vendoring the patched source into your own project repo

## Files

| File | Description |
|---|---|
| `model.py` | Patched `KModel` — local-first model and config loading |
| `pipeline.py` | Patched `KPipeline` — local-first voice file loading |
