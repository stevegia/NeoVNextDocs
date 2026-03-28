# WAN Video Generation Smoke Test

## Purpose

The WAN smoke test page (`Admin > WAN Smoke`) is an **admin-only tool** for
validating WAN (Wan Ai) video generation on Cloud Run GPU workers.

**This is not core product functionality.** NeoVLab is a media management and
processing platform, not a video generation UI.

This page exists because:

- WAN video generation infrastructure will be important for future planned
  features in NeoVLab's processing pipeline.
- After deploying changes to the GPU worker, a quick end-to-end check confirms
  that WAN models load correctly, the ComfyUI queue processes the job, and
  artifacts are stored.
- The existing settings smoke test (SDXL) validates image generation. The WAN
  smoke test specifically exercises video generation node types.

---

## Workflows Available

| Mode | Model | Use Case |
|------|-------|----------|
| T2V 1.3B bf16 | `wan2.1_t2v_1.3B_bf16.safetensors` | Fastest pipeline check, lower quality |
| T2V 14B fp8 | `wan2.1_t2v_14B_fp8_scaled.safetensors` | Quality text-to-video |
| I2V 14B fp8 | `wan2.1_i2v_480p_14B_fp8_scaled.safetensors` | Image-to-video (needs start frame) |

All workflows use:
- Text encoder: `umt5_xxl_fp8_e4m3fn_scaled.safetensors`
- VAE: `wan_2.1_vae.safetensors`
- I2V additionally: CLIP vision encoder + `LoadImage` node (expects `example.png` in ComfyUI input dir)

---

## Prerequisites Before Running

1. **Models in GCS** -- WAN diffusion models, text encoders, and VAE must be
   present in the worker model volume (`gs://neovnext-neovlab-models/`).
   - Path: `diffusion_models/wan/wan2.1_t2v_1.3B_bf16.safetensors` (etc.)
   - Text encoder path: `text_encoders/umt5_xxl_fp8_e4m3fn_scaled.safetensors`
   - VAE path: `vae/wan_2.1_vae.safetensors`

2. **Cloud Run GPU worker running** -- Check `Admin > Settings > GPU Status`.
   Cold start is ~2-3 min before inference begins.

3. **For I2V** -- Upload a start frame image to the ComfyUI worker's
   `/input/example.png`. This requires accessing the worker container directly
   or pre-seeding the image into the Docker image.

---

## Expected Timing

| Mode | Approx. Duration |
|------|-----------------|
| T2V 1.3B, 9 frames, 20 steps | 3-8 min (inc. cold start) |
| T2V 14B, 17 frames, 20 steps | 10-20 min |
| I2V 14B, 17 frames, 20 steps | 10-20 min |

---

## Output

Successful runs produce `.webp` animated images (multi-frame WebP) or `.webm`
video files depending on the SaveNode used. These appear under `Admin >
Artifacts` in the `generated` domain and can be downloaded directly from the
WAN Smoke Test page.

---

## Limitations

- This page does not validate WAN 2.2 models (i2v, t2v, s2v, fun-inpaint,
  fun-camera). Those require separate workflow definitions.
- The I2V workflow requires `example.png` to be manually seeded into the
  worker's ComfyUI input directory.
- There is no streaming progress on this page -- status is polled every 5 sec.
  For detailed logs, use `Admin > Jobs` and select the submitted job ID.

---

## Related Docs

- `docs/runbook-comfyui-cloud-run-debugging.md` -- Worker debugging runbook
- `docs/runbook-gcp-gpu.md` -- GCP GPU infrastructure runbook
- Local WAN models: `Z:\comfy\ComfyUI\models\`
- NeoComfy reference implementation: `Z:\NeoComfy` (C# + React studio with
  full I2V/T2V pipeline that this smoke test is modelled on)


---

## Canonical Template Smoke-Test Procedures (T052 / US2)

### Pre-conditions
- ComfyUI worker is running and reachable at `http://127.0.0.1:8188`
- `IO_CACHE_DIR` volume is mounted and writable
- PostgreSQL cluster DB is up and worker is registered

### sd_png Template Test
1. Stage a source image (for img2img) or empty manifest (for txt2img):
   ```
   POST /api/v2/jobs
   { "jobType": "comfyui", "payloadJson": { "template_type": "sd_png", "workflow": {...}, "output_node_ids": [9] } }
   ```
2. Poll `GET /api/v2/jobs/{jobId}` until `status == "synced"`.
3. Call `GET /api/v2/artifacts/by-job/{jobId}`.
4. Assert: exactly one artifact with `isPrimary=true` and `artifactType="generated_image"`.
5. Assert: file extension ends in `.png`.

### image_to_video_mp4 Template Test (Wan 2.2 I2V)
1. Stage source frame to IO cache; record its `cachePath`.
2. Enqueue job with manifest `{ "source.png": "<cachePath>" }` and
   `template_type = "image_to_video_mp4"`.
3. Poll until `status == "synced"` (expected ~3-5 min on 24 GB GPU).
4. Call `GET /api/v2/artifacts/by-job/{jobId}`.
5. Assert: primary artifact has `artifactType="generated_video"` and content type `video/mp4`.
6. Assert: `fileSizeBytes > 0`.
7. Open the artifact via `GET /api/v2/artifacts/{artifactId}/stream` and confirm
   the response plays in MPC-HC or ffprobe without errors.

### start_end_frame_mp4 Template Test (Wan 2.2 Keyframe)
1. Stage start frame and end frame to IO cache.
2. Enqueue job with manifest `{ "start.png": "<startCachePath>", "end.png": "<endCachePath>" }`
   and `template_type = "start_end_frame_mp4"`.
3. Poll until `status == "synced"`.
4. Assert primary artifact `artifactType="generated_video"`, `video/mp4`.
5. Verify motion interpolation: first and last frames of the resulting video
   should be visually similar to the start/end inputs.

### FR-040 Failure Test
1. Enqueue a ComfyUI job with a deliberately broken workflow (no output node connected).
2. Assert the job transitions to `status = "failed"` (not `"complete"` / `"synced"`).
3. Assert `GET /api/v2/artifacts/by-job/{jobId}` returns an empty list.
4. Confirm no partial artifacts are written to the cold local store.
