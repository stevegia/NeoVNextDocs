# Python Handler Compatibility Notes

**Base image:** `runpod/comfyui:cuda13.0`
**Python:** 3.12 | **PyTorch:** 2.10+cu130 | **torchaudio:** 2.10 | **torchaudio:** 2.10

Each handler runs in an isolated venv under `/venvs/<name>/` with `--system-site-packages`
to inherit system torch/numpy/transformers without reinstalling them.

---

## Diarization (pyannote.audio)

### Issue 1 — torch 2.10 weights_only=True (fixed in v1.0.6)

**Symptom:**
```
_pickle.UnpicklingError: ... OmegaConf ListConfig/DictConfig not in safe globals
```

**Cause:** PyTorch 2.6+ defaults `torch.load(weights_only=True)`. pyannote checkpoint
files serialize OmegaConf objects (`ListConfig`, `DictConfig`) which are not in torch's
safe unpickling allowlist.

**Fix:** Upgraded from `pyannote.audio==3.1.1` to `pyannote.audio>=3.4,<4`, which has
partial torch 2.6+ compat. Added `torch.serialization.add_safe_globals([DictConfig, ListConfig])`
in `diarization.py:init()` as belt-and-suspenders.

**Why not pyannote 4.x?** 4.x hard-pins `torch==2.8.0`, incompatible with our system torch 2.10.

---

### Issue 2 — torchaudio 2.10 removed AudioMetaData (fixed in v1.0.7)

**Symptom:**
```
AttributeError: module 'torchaudio' has no attribute 'AudioMetaData'
  File ".../pyannote/audio/core/io.py", line 60
```

**Cause:** `torchaudio.AudioMetaData` was deprecated in torchaudio 2.8 and removed in 2.9.
pyannote.audio 3.x references it in `pyannote/audio/core/io.py` as a return-type annotation
and as a `list_audio_backends()` call. Python evaluates annotations at module import time,
causing an immediate crash before any model loading.

**Upstream status:** Open issue [pyannote/pyannote-audio#1952](https://github.com/pyannote/pyannote-audio/issues/1952).
No 3.x backport planned. 3.x is effectively abandoned in favor of 4.x.

**Fix:** Post-install `sed` patches applied in the Dockerfile immediately after `pip install`:

```dockerfile
&& PYANNOTE_IO=/venvs/diarization/lib/python3.12/site-packages/pyannote/audio/core/io.py \
&& sed -i 's/-> torchaudio\.AudioMetaData:/-> object:/g' "$PYANNOTE_IO" \
&& sed -i "s/torchaudio\.list_audio_backends()/getattr(torchaudio, 'list_audio_backends', lambda: ['sox_io'])()/g" "$PYANNOTE_IO"
```

Patch 1: replaces `-> torchaudio.AudioMetaData:` return annotation with `-> object:`
Patch 2: wraps `torchaudio.list_audio_backends()` in a `getattr` with a default so it
doesn't crash if the function is missing.

The patches are idempotent — if pyannote.audio ever fixes this upstream, the sed patterns
won't match and the build continues cleanly.

**Future upgrade path:** If moving to pyannote 4.x, remove both sed lines. Verify
`torchcodec` builds against CUDA 13.0 first (4.x switched from torchaudio to torchcodec).

---

## handler_server.py — FastAPI lifespan (fixed in v1.0.6)

**Symptom:**
```
DeprecationWarning: on_event is deprecated, use lifespan event handlers instead.
```

**Fix:** Replaced `@app.on_event("startup")` with `@asynccontextmanager` lifespan pattern:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    _handler_module = get_handler(HANDLER_NAME)
    yield

app = FastAPI(lifespan=lifespan)
```

---

## Version matrix

| Image | pyannote.audio | torch | torchaudio | Status |
|-------|---------------|-------|------------|--------|
| v1.0.5 | 3.1.1 | 2.10+cu130 | 2.10 | Crashes: weights_only |
| v1.0.6 | >=3.4,<4 | 2.10+cu130 | 2.10 | Crashes: AudioMetaData |
| v1.0.7 | >=3.4,<4 + sed patch | 2.10+cu130 | 2.10 | Working |
