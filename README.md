# How to run Kokoro TTS offline without triggering the Network Issue 

Patch for running [Kokoro TTS](https://github.com/hexgrad/kokoro) fully offline by loading model weights
and config from a local directory instead of downloading from HuggingFace Hub on every run.

## Problem

By default, the Kokoro TTS library unconditionally calls `hf_hub_download()` at runtime — even if the
model files are already present locally. This causes inference to fail in offline or air-gapped environments
(e.g. Nvidia Jetson devices without internet access).

The affected source file is:
- `kokoro/model.py` — downloads `config.json` and `.pth` model weights from HuggingFace on every init

## Solution

Patch `model.py` inside your virtual environment to prioritize a local model directory,
with HuggingFace download demoted to a fallback only when a file is genuinely missing.

### Key Changes

| File | What was changed |
|---|---|
| `model.py` | Added `local_model_dir` param to `KModel.__init__`; loads `config.json` and `.pth` from local path |

`hf_hub_download` is now a lazy import inside a fallback block only, so the network is never touched
if all files are present locally.

## How to Apply

### 1. Download model files from HuggingFace (one-time, while online)

Download both `kokoro-v1_0.pth` and `config.json` from the
[Kokoro HuggingFace page](https://huggingface.co/hexgrad/Kokoro-82M/tree/main)
and place them in your local model directory.

### 2. Locate the Kokoro source file in your venv

```bash
find .venv -path "*/kokoro/model.py"
```

### 3. Replace with the patched file

Copy the patched `model.py` from this repo into the path found above:

```bash
cp model.py /path/to/.venv/lib/python3.10/site-packages/kokoro/model.py
```

### 4. Update the local model path

In `model.py`, update `DEFAULT_LOCAL_MODEL_DIR` to match your setup:

```python
DEFAULT_LOCAL_MODEL_DIR = '/your/path/to/models/tts_kokoro'
```

### 5. Verify offline inference

Disconnect from the network and run your TTS pipeline. You should see log output like:

```
DEBUG | Loading config from local path: /your/path/models/tts_kokoro/config.json
DEBUG | Loading model weights from local path: /your/path/models/tts_kokoro/kokoro-v1_0.pth
```

## Notes

- This patch was tested on **Kokoro 82M v1.0** (`hexgrad/Kokoro-82M`) on an **Nvidia Jetson Orin Nano**
  running **Python 3.10**
- If you upgrade the Kokoro package, re-apply the patch as the source file will be overwritten
- To avoid re-patching on every upgrade, consider vendoring the patched source into your own project repo

## Files

| File | Description |
|---|---|
| `model.py` | Patched `KModel` — local-first model and config loading |




