# Model Requirements Runbook

Reference for all models used by the ComfyUI workflows in NeoVNext. Covers what is currently
deployed on the RunPod volume and what must be added before new workflow presets can execute.

---

## Storage Layout

All models are stored under the RunPod persistent volume at `/workspace/comfyui/`. ComfyUI's model
directories are mapped as follows:

| Model type          | Volume path                            |
|---------------------|----------------------------------------|
| Diffusion UNets     | `/workspace/comfyui/unet/wan/`         |
| Text encoders       | `/workspace/comfyui/text_encoders/`    |
| VAE                 | `/workspace/comfyui/vae/`              |
| CLIP vision         | `/workspace/comfyui/clip_vision/`      |
| LoRAs               | `/workspace/comfyui/loras/`            |
| SD/SDXL checkpoints | `/workspace/comfyui/checkpoints/`      |
| IP-Adapter weights  | `/workspace/comfyui/ip-adapter/`       |

---

## Currently Deployed Models

These models are referenced by the existing WAN presets in `appsettings.json`.

### WAN Diffusion UNets (`/workspace/comfyui/unet/wan/`)

| Filename | Preset | Notes |
|---|---|---|
| `wan2.1_t2v_1.3B_bf16.safetensors` | `t2v-1.3b` | Fast T2V; bf16 precision |
| `wan2.1_t2v_14B_fp8_scaled.safetensors` | `t2v-14b` | Quality T2V; fp8_e4m3fn |
| `wan2.1_i2v_480p_14B_fp8_scaled.safetensors` | `i2v-14b`, `i2v-14b-cast`, `i2v-14b-multi-cast` | I2V; fp8_e4m3fn |
| `wan2.2_i2v_high_noise_14B_fp8_scaled.safetensors` | `transform-22`, `transform-22-nsfw`, `transform-22-nsfw-fast` | Transform dual-pass (high noise) |
| `wan2.2_i2v_low_noise_14B_fp8_scaled.safetensors` | `transform-22`, `transform-22-nsfw`, `transform-22-nsfw-fast` | Transform dual-pass (low noise) |
| `wan22EnhancedNSFWCameraPrompt_nsfwFASTMOVEV2FP8H.safetensors` | `transform-22-nsfw`, `transform-22-nsfw-fast` | NSFW camera high-noise model |
| `wan22EnhancedNSFWCameraPrompt_nsfwFASTMOVEV2FP8L.safetensors` | `transform-22-nsfw`, `transform-22-nsfw-fast` | NSFW camera low-noise model |

### LightX2V LoRAs (`/workspace/comfyui/loras/`)

| Filename | Preset | Notes |
|---|---|---|
| `wan2.2_i2v_lightx2v_4steps_lora_v1_high_noise.safetensors` | `transform-22`, `transform-22-nsfw-fast` | 4-step distillation, high-noise pass |
| `wan2.2_i2v_lightx2v_4steps_lora_v1_low_noise.safetensors` | `transform-22`, `transform-22-nsfw-fast` | 4-step distillation, low-noise pass |

### Text Encoder (`/workspace/comfyui/text_encoders/`)

| Filename | Used by | Notes |
|---|---|---|
| `umt5_xxl_fp8_e4m3fn_scaled.safetensors` | All WAN presets | WAN 2.1/2.2 text encoder; `WanPresets.SharedModels.textEncoder` |

### VAE (`/workspace/comfyui/vae/`)

| Filename | Used by | Notes |
|---|---|---|
| `wan_2.1_vae.safetensors` | All WAN presets | `WanPresets.SharedModels.vae` |

### CLIP Vision (`/workspace/comfyui/clip_vision/`)

| Filename | Used by | Notes |
|---|---|---|
| `clip_vision_h.safetensors` | All I2V and Transform presets | `WanPresets.SharedModels.clipVision`; also required by IP-Adapter workflows |

### SDXL Checkpoints (`/workspace/comfyui/checkpoints/`)

| Filename | Preset | Notes |
|---|---|---|
| `animagineXL_v31.safetensors` | `animagine-xl-v31` | Anime-style SDXL |
| `ponyDiffusionV6XL_v6StartWithThisOne.safetensors` | `pony-diffusion-v6` | Pony Diffusion V6 XL |
| `ponyRealism_V23ULTRA.safetensors` | `pony-realism-v23` | Pony Realism V2.3 |

---

## New Models Required

These models must be downloaded and placed on the RunPod volume before the `i2v-14b-cast`
and `i2v-14b-multi-cast` presets can execute.

### IP-Adapter Weights (`/workspace/comfyui/ip-adapter/`)

| Filename | Purpose | Status |
|---|---|---|
| `ip-adapter-plus_sdxl_vit-h.safetensors` | IP-Adapter conditioning for cast face reference | **REQUIRED — not yet on volume** |

**Download:**

```
# Hugging Face repo: h94/IP-Adapter
# File path within repo: sdxl_models/ip-adapter-plus_sdxl_vit-h.safetensors
# Direct URL:
https://huggingface.co/h94/IP-Adapter/resolve/main/sdxl_models/ip-adapter-plus_sdxl_vit-h.safetensors
```

**On the RunPod pod:**

```bash
mkdir -p /workspace/comfyui/ip-adapter
cd /workspace/comfyui/ip-adapter
wget -O ip-adapter-plus_sdxl_vit-h.safetensors \
  "https://huggingface.co/h94/IP-Adapter/resolve/main/sdxl_models/ip-adapter-plus_sdxl_vit-h.safetensors"
```

**Notes:**

- Approximate file size: ~860 MB.
- Works with the existing `clip_vision_h.safetensors` encoder already on the volume.
- The `ip-adapter-plus` variant (rather than base `ip-adapter`) provides stronger facial feature
  conditioning and is recommended for cast reference workflows.
- The model filename `ip-adapter-plus_sdxl_vit-h.safetensors` is the value referenced in
  `WanPresets.Workflows[i2v-14b-cast].models.ipAdapter` and
  `WanPresets.Workflows[i2v-14b-multi-cast].models.ipAdapter`.

---

## IP-Adapter ComfyUI Node Requirements

The cast workflow presets require the following ComfyUI custom nodes to be installed:

```
ComfyUI-IPAdapter-plus (https://github.com/cubiq/ComfyUI_IPAdapter_plus)
```

This provides the `IPAdapterModelLoader`, `PrepImageForClipVision`, and `IPAdapterApply` nodes
used by the cast workflow JSON.

Install on the RunPod pod:

```bash
cd /workspace/comfyui/custom_nodes
git clone https://github.com/cubiq/ComfyUI_IPAdapter_plus.git
pip install -r ComfyUI_IPAdapter_plus/requirements.txt
```

---

## Multi-Cast Identity Preservation Caveat

The `i2v-14b-multi-cast` preset uses dual IP-Adapter conditioning to reference two persons.
IP-Adapter + WAN provides **style-level face similarity**, not exact identity swap. The output
will feature faces that resemble the reference crops, but perfect identity preservation would
require additional models such as InstantID or ReActor. This is documented in
`docs/planning/feature-2-comfyui-expansion/DESIGN.md`.
