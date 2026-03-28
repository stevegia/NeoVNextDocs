# Feature 2 -- ComfyUI Workflow Expansion + Cast-Driven Smoke Tests

## Status Quo

### Current ComfyUI Capabilities

The system has a working ComfyUI integration with:
- **Python handler** (`src/workers/handlers/comfyui.py`): Submits workflow JSON to local ComfyUI server, polls for completion, downloads output artifacts.
- **WAN presets** in `appsettings.json`: T2V 1.3B, T2V 14B, I2V 14B, Transform WAN 2.2 (standard, NSFW, NSFW-fast).
- **SD presets**: Animagine XL checkpoint for text-to-image.
- **Frontend smoke tests**: SmokeTestsPage with tabs for ComfyUI, SD, and WAN smoke tests.
- **Workflow construction**: The frontend builds ComfyUI workflow JSON from presets and submits via the API.

### Current Model Set (on RunPod volume)

From the WAN presets config:
- `wan2.1_t2v_1.3B_bf16.safetensors` (T2V, fast)
- `wan2.1_t2v_14B_fp8_scaled.safetensors` (T2V, quality)
- `wan2.1_i2v_480p_14B_fp8_scaled.safetensors` (I2V, single start frame)
- `wan2.2_i2v_high_noise_14B_fp8_scaled.safetensors` (Transform, high noise)
- `wan2.2_i2v_low_noise_14B_fp8_scaled.safetensors` (Transform, low noise)
- `wan22EnhancedNSFWCameraPrompt_nsfwFASTMOVEV2FP8H.safetensors` (NSFW high)
- `wan22EnhancedNSFWCameraPrompt_nsfwFASTMOVEV2FP8L.safetensors` (NSFW low)
- LightX2V 4-step LoRAs for fast generation
- `umt5_xxl_fp8_e4m3fn_scaled.safetensors` (text encoder)
- `wan_2.1_vae.safetensors` (VAE)
- `clip_vision_h.safetensors` (CLIP vision for I2V)
- `animagineXL_v31.safetensors` (SDXL checkpoint)

### NeoComfy Reference

NeoComfy's `ComfyUiService.cs` has working implementations of:
- `QueueT2VAsync` -- text-to-video with configurable model
- `QueueI2VAsync` -- image-to-video with start frame
- `QueueSDXLAsync` -- text-to-image with SDXL checkpoint
- `GetStatusAsync` -- poll ComfyUI history for completion
- Venice AI video generation as an alternative backend
- Transform workflows with dual-frame (start+end) support

The `VeniceModelCatalog.cs` has a comprehensive registry of Venice AI video models including Wan 2.5/2.6, Longcat, Kling, Veo, PixVerse, and Vidu.

---

## New Capabilities to Implement

### 1. Cast-Member-Driven I2V Workflow (IP-Adapter Reference)

**Goal:** Use a cast member's face image as a reference/conditioning input to guide I2V generation so the output features that person.

**Approach -- IP-Adapter + CLIP Vision:**
ComfyUI supports IP-Adapter nodes that use CLIP image embeddings to condition generation. The WAN I2V pipeline already uses `clip_vision_h.safetensors` for start-frame conditioning. We can extend this by:

1. Using the person's best face crop (or generated portrait) as an IP-Adapter reference image
2. Adding an `IPAdapterApply` node to the workflow that injects the face embedding alongside the text prompt
3. The I2V workflow becomes: text prompt + reference face image -> video featuring that person

**Required Models (to add to volume):**
- `ip-adapter_sdxl_vit-h.safetensors` or `ip-adapter-plus_sdxl_vit-h.safetensors` -- IP-Adapter weights
- These work with the existing `clip_vision_h.safetensors` encoder

**Workflow JSON Structure:**
```
New nodes to add to existing I2V workflow:
- IPAdapterModelLoader -> loads IP-Adapter model
- PrepImageForClipVision -> preprocesses person face crop
- IPAdapterApply -> conditions the model with face embedding
```

**New Preset:**
```json
{
  "id": "i2v-14b-cast",
  "name": "I2V -- 14B Cast Member Reference",
  "category": "i2v-cast",
  "description": "Image to video conditioned on a cast member's face via IP-Adapter. Start frame + person face reference.",
  "models": {
    "unet": "wan/wan2.1_i2v_480p_14B_fp8_scaled.safetensors",
    "unetWeightDtype": "fp8_e4m3fn",
    "ipAdapter": "ip-adapter-plus_sdxl_vit-h.safetensors"
  },
  "defaults": { "steps": 20, "cfg": 6.0, "sampler": "euler", "scheduler": "beta", "ipAdapterStrength": 0.7 }
}
```

### 2. Multi-Character Workflow

**Goal:** Generate a video featuring 2 cast members interacting based on a text prompt.

**Approach:**
- Use dual IP-Adapter conditioning: two separate IP-Adapter nodes, each with a different person's face crop
- The text prompt describes the scene/interaction
- IP-Adapter strength per character controls influence

**Limitations:** Current IP-Adapter + WAN may not perfectly swap faces. This is more about "style conditioning" -- the output will feature faces *similar* to the references. Perfect face identity preservation would require additional models (e.g., InstantID, ReActor). Document this honestly.

**New Preset:**
```json
{
  "id": "i2v-14b-multi-cast",
  "name": "I2V -- 14B Multi-Cast (2 persons)",
  "category": "i2v-cast",
  "description": "Image to video conditioned on two cast members via dual IP-Adapter. Experimental.",
  "models": {
    "unet": "wan/wan2.1_i2v_480p_14B_fp8_scaled.safetensors",
    "unetWeightDtype": "fp8_e4m3fn",
    "ipAdapter": "ip-adapter-plus_sdxl_vit-h.safetensors"
  },
  "defaults": { "steps": 25, "cfg": 6.0, "sampler": "euler", "scheduler": "beta", "ipAdapterStrength1": 0.6, "ipAdapterStrength2": 0.6 }
}
```

### 3. Audio-to-Video Research

**Current state:** None of our local ComfyUI models support audio-to-video directly. WAN 2.1/2.2 are video-only diffusion models.

**What exists in the ecosystem:**
- **Venice AI** has Wan 2.5/2.6 with `SupportsAudio: true` -- these models can generate video WITH audio, but this is output audio, not audio-driven generation
- **ComfyUI audio reactive nodes** exist (e.g., `ComfyUI-AudioReactive`) that can drive video generation from audio beat/amplitude analysis, but these are visual-rhythm sync, not speech-to-lip-sync
- **Lip sync models** (SadTalker, Wav2Lip, MuseTalk) can animate a face image from audio -- this IS audio-to-video in the meaningful sense
- **MuseTalk** is the most promising: it can take a face image + audio clip and produce a talking-head video

**Recommendation:** Audio-to-video as "make a video from an audio file" is not directly supported. However, **audio-driven face animation** (MuseTalk) IS achievable and highly relevant to the cast pipeline (person face + cloned voice audio -> talking head video). This should be a future feature after Features 1+3 are working.

For now, document that Venice AI's Wan 2.5/2.6 can generate videos with audio output, and add Venice as an alternative video generation backend alongside local ComfyUI.

### 4. Venice AI as Alternative Video Backend

NeoComfy already has a full Venice API integration. Port the `VeniceModelCatalog` pattern into NeoVNext:

- Add Venice video generation as an alternative to local ComfyUI
- The API is OpenAI-compatible: `POST https://api.venice.ai/api/v1/chat/completions` for LLM, `POST https://api.venice.ai/api/v1/images/generations` for images, and `POST https://api.venice.ai/api/v1/video/generations` for video
- Venice supports uncensored models (Wan 2.5/2.6, Longcat) which are important for this project
- Venice keys are stored in the settings DB (Feature 3 handles the settings UI)

---

## Implementation Plan

### 1. New WAN Presets in appsettings.json

Add to `WanPresets.Workflows`:
- `i2v-14b-cast`: I2V with IP-Adapter cast member reference
- `i2v-14b-multi-cast`: I2V with dual IP-Adapter (2 persons)

### 2. ComfyUI Workflow Builder Functions

Create new workflow builder functions in the frontend or backend that construct the ComfyUI API workflow JSON for cast-driven workflows. These need to:
- Accept `personImagePath` (or `personImageUrl`) parameters
- Stage the person's face crop image into ComfyUI's input directory (use existing `InputStagingService`)
- Build the workflow JSON with IP-Adapter nodes referencing the staged image

**Files:**
- `src/frontend/src/lib/comfyWorkflows.ts` -- add `buildCastI2VWorkflow()` and `buildMultiCastI2VWorkflow()`
- Or, add workflow builder methods to the C# backend if workflow construction should be server-side

### 3. API Endpoints for Cast-Driven Generation

```
POST /api/comfy/cast-i2v
  Body: { personId, startFramePath, prompt, presetId?, settings? }
  - Resolves person's face crop image
  - Stages it as ComfyUI input
  - Builds and submits workflow
  - Returns job ID for polling

POST /api/comfy/multi-cast-i2v
  Body: { personIds: [id1, id2], prompt, presetId?, settings? }
  - Same pattern with two person face crops
```

### 4. Smoke Test UI Extensions

Add to the SmokeTestsPage:
- **New tab: "Cast Workflows"** -- requires at least one person in the DB
- Person selector dropdown(s) (fetches from /api/persons)
- Prompt input
- Start frame image upload (optional)
- IP-Adapter strength slider
- Preset selector (filtered to i2v-cast category)
- Generate button -> submits to /api/comfy/cast-i2v
- Output display (reuses existing smoke test output viewer)

### 5. Venice AI Video Backend (Stretch)

Add a `VeniceVideoService` to the C# backend:
- `GenerateVideoAsync(prompt, model, options)` -- submits to Venice API
- Poll for completion
- Download result
- Store as artifact

Add Venice models to a new config section or to the existing presets:
```json
"VenicePresets": {
  "ApiBaseUrl": "https://api.venice.ai/api/v1",
  "Models": [
    { "id": "wan-2.6-text-to-video", "name": "Wan 2.6 (Venice)", "category": "t2v-venice", ... }
  ]
}
```

### 6. Model Requirements Runbook

Create `docs/runbook/models.md` documenting:
- All models currently on the RunPod volume
- New models needed for IP-Adapter workflows
- Download URLs and expected file sizes
- Venice AI models available (no local storage needed)

---

## Files to Modify/Create

### Backend (C#)
- `src/backend/NeoVLab.Api/appsettings.json` -- add new presets
- `src/backend/NeoVLab.Api/Controllers/ComfyInputController.cs` -- extend for cast image staging
- NEW: `src/backend/NeoVLab.Api/Controllers/CastWorkflowController.cs` -- cast-driven generation endpoints
- NEW: `src/backend/NeoVLab.Api/Services/VeniceVideoService.cs` (stretch)

### Frontend
- `src/frontend/src/pages/SmokeTestsPage.tsx` -- add Cast Workflows tab
- NEW: `src/frontend/src/pages/CastSmokeTestPage.tsx` -- cast workflow smoke test UI
- `src/frontend/src/lib/comfyWorkflows.ts` or similar -- workflow JSON builders
- `src/frontend/src/api/client.ts` -- new API calls

### Docs
- NEW: `docs/runbook/models.md` -- model requirements documentation

---

## Dependency Map

| Step | Depends On | Notes |
|------|-----------|-------|
| 1. New presets | Nothing | Config-only change |
| 2. Workflow builders | Step 1 | Frontend or backend |
| 3. Cast API endpoints | Step 2, Feature 1 (persons exist) | Stubs person face with placeholder if Feature 1 not done |
| 4. Smoke test UI | Steps 1, 2, 3 | |
| 5. Venice backend | Feature 3 (API key settings) | Stretch goal |
| 6. Runbook | Steps 1-4 | Documentation |

## Integration with Other Features

- **Feature 1** provides `persons` with face crops and descriptions. Feature 2 uses the face crop as IP-Adapter input. If Feature 1 is not complete, use a placeholder image upload in the smoke test UI.
- **Feature 3** provides Venice AI API key management and LLM service for prompt enhancement. Feature 2's Venice video backend depends on having a Venice API key configured.
