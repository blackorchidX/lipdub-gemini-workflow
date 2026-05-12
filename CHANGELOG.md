# Changelog

All notable changes to the Lipdub‑Gemini workflow.

## [1.2.0] — 2026‑05‑12

### Added
- **`LipDub-Gemini-UI.json`** — full ComfyUI UI‑format workflow with 7 hand‑authored colored groups (`Load Models`, `Load Original Video`, `Stage 1`, `Gemini`, `Stage 2`, `Decode`, `Output`), node positions, and link metadata. Loading this in ComfyUI gives the original visual layout back.
- The existing API‑format JSON is unchanged and remains the source of truth for programmatic use.

## [1.1.0] — 2026‑05‑12

### Fixed
- **Gemini API HTTP 413 "Payload Too Large" error.** The Gemini node was receiving all source frames as individual base64 PNGs (the first 10 via URL upload, the rest inlined into the request JSON), inflating the body past the API's ~20 MB cap.
  - Added node `5025` `CreateVideo (Gemini Input)` that packs the 360×360 resized frames + source audio into an H.264 MP4.
  - Rewired the Gemini node to consume `video` (one small MP4) instead of `images + audio` (~hundreds of large PNGs).
  - Request body now scales with H.264 compression instead of base64 PNG, going from ~20‑30 MB to ~1‑3 MB.

### Added
- **Automatic length detection from source video.** No more manual `frame_load_cap` editing.
  - `VHS_LoadVideo.frame_load_cap` set to `0` (load all frames).
  - New node `5026` `Snap to LTX‑valid length` (`easy simpleMath`, formula `floor((a-1)/8)*8+1`) consumes VHS's `frame_count` output.
  - LTX `EmptyLTXVLatentVideo.length` and `LTXVEmptyLatentAudio.frames_number` now read from the snapped value.
  - Result: drop any source video in → workflow runs at the largest valid `N×8+1` length ≤ source length.

### Changed
- **Output encoding upgraded from `SaveVideo` (auto codec, no quality knobs) to `VHS_VideoCombine`.**
  - Codec: `h264-mp4`, pixel format: `yuv420p`, CRF: `17` (near visually lossless), metadata preserved.
  - Removed obsolete `CreateVideo` node in the output path.
  - The previous `SaveVideo` defaulted to a very conservative bitrate, causing perceptible softness; CRF 17 keeps detail intact.
- **Stage‑2 upscale refinement bumped from 3 → 5 denoising steps.**
  - Sigmas: `0.909375, 0.825, 0.725, 0.55, 0.421875, 0.0` (was `0.909375, 0.725, 0.421875, 0.0`).
  - More detail recovery on the 1920×1088 pass at ~65 % extra stage‑2 compute (a few seconds per run).
- **Tiled VAE decode overlap raised from `6` → `8` latent cells** (≈256 px at the 32× VAE), the maximum the node permits. Reduces visible tile seams.

### Organized
- **Group prefix tags added to every node's `_meta.title`.** Categories: `Input`, `Gemini Prompt`, `Text Encode`, `Models/LoRAs`, `Stage 1 - Base`, `Stage 2 - Upscale`, `Decode`, `FPS/utils`, `Output`. Makes the graph self‑documenting even after auto‑layout.

### Removed
- `4849` `CreateVideo` — replaced by direct images/audio wiring into `VHS_VideoCombine`.
- `4988` `PrimitiveInt` (manual frame count) — replaced by automatic detection via VHS `frame_count` → snap math.

### Node delta

| Added | Removed | Repurposed |
|---|---|---|
| `5025` Create Video (Gemini Input) | `4849` CreateVideo | `4852` SaveVideo → VHS_VideoCombine |
| `5026` Snap to LTX‑valid length | `4988` PrimitiveInt cap | |

Net node count: **47** (was 47).
