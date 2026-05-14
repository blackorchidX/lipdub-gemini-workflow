# Changelog

All notable changes to the Lipdub‑Gemini workflow.

## [1.8.0] — 2026‑05‑14

### Polished
- **Canvas layout overhaul.** The workflow has been re‑arranged inside ComfyUI for a clean left‑to‑right data flow with consistent column alignment per group, no zero functional change. Layout convention going forward:
  - `Load Models` (blue) — top‑left; loaders in a left column, perf‑patch chain in a right column.
  - `Load Original Video` (blue) — bottom‑left.
  - `Stage 1` (gold `#b58b2a`) — center‑left, three columns: raw inputs / concat & mask / switches & sampler.
  - `Gemini` (purple `#a1309b`) — center, three columns: prompt source / encoders / output (Gemini & manual).
  - `Stage 2` (blue) — center‑right.
  - `Decode` (blue) — right.
  - `Output` (purple) — far‑right.
  - Vertical step ~100–150 px between sibling nodes; horizontal column step ~400–500 px.

### Resynced
- `Lipdub-Gemini-API.json` re‑exported from ComfyUI to match the new layout. Note: Decode group node IDs were renumbered by ComfyUI's exporter (`4848 / 4995 / 5015 → 5073 / 5074 / 5075`); UI workflow retains its own ID space. Both files are internally consistent.

### Aesthetic policy (for future patches)
- New nodes will be placed **inside the appropriate group bounds** at the next free row of the matching column.
- New groups require explicit approval before being created.
- The colour palette (blue = processing, gold = main work, purple = user‑facing surfaces) is preserved.

## [1.7.0] — 2026‑05‑14

### Added
- **Performance tweak chain** after the IC‑LoRA loader, inspired by RuneXX's *Just‑Dub‑It* workflow. All three are KJNodes (already installed). The chain is `5012 IC‑LoRA → 5037 → 5038 → 5039 → CFGGuiders & SetMaskByTime`.
  - **`5037 PathchSageAttentionKJ`** — set to `sage_attention="disabled"` (no‑op placeholder). Flip to `auto` after `pip install sageattention` for a Sage Attention speed/VRAM win.
  - **`5038 LTXVChunkFeedForward`** — `chunks=2, dim_threshold=4096`. Splits the FFN activation tensor in 2 across the sequence dim when it exceeds the threshold; pure‑PyTorch VRAM saver, no extra dependencies, mathematically equivalent to non‑chunked.
  - **`5039 LTX2AttentionTunerPatch`** — `blocks=""` (all), all 4 attention scales `= 1.0`, `triton_kernels=False`. Replaces the LTX2 forward pass with an alternate implementation that claims VRAM reductions at neutral scales; experimental but reversible.

### Tunable
- Each patcher can be muted (`Ctrl+M`) or unwired to revert. The chain is straight‑line, so removing any one node and reconnecting its neighbours is trivial.
- Once you `pip install sageattention` (and ideally `triton`), set `5037.sage_attention = "auto"` for the largest speed/VRAM gain; optionally also set `5039.triton_kernels = True`.

### Note
These patchers are marked **EXPERIMENTAL** by KJNodes. With the conservative defaults here, output should be identical or near‑identical to v1.6 — they're aimed at reducing peak VRAM and (with extra packages) speeding up inference. If you observe artifacts on a specific clip, mute the patchers and report which ones.

## [1.6.0] — 2026‑05‑14

### Changed
- **Negative prompt upgraded** for LipDub‑specific failure modes. Replaced the short generic negative `"pc game, console game, video game, cartoon, childish, ugly"` with a comprehensive 30‑token string covering:
  - Audio failures (distorted/saturated/muffled/loud sound)
  - Lip‑sync and face failures (deformed/asymmetrical features, disfigured/blurry teeth, mouth out of sync)
  - Overlay artifacts (text, subtitles, logo, watermark, signature)
  - Generic LTX quality failures (low quality, blurry, pixelated, compression/jpeg artifacts, glitches)
  - Original aesthetic preferences kept (pc game, console game, video game, cartoon, childish, ugly)
- Both `LipDub-Gemini-UI.json` and `Lipdub-Gemini-API.json` updated. Adapted from RuneXX's *Just‑Dub‑It* negative prompt with additions targeting LipDub's specific failure modes (audio and mouth/teeth).

## [1.5.1] — 2026‑05‑13

### Fixed
- **Prompt‑source toggle silently falling back to Manual.** ComfyUI's UI re‑serializes INT widget values as JSON strings on save (e.g. `'1'` instead of `1`), and `easy textSwitch.switch()` uses a strict `input == 1` comparison: in Python, `'1' == 1` is `False`, so the switch was returning `text2` (Manual) anytime the workflow had been saved through the canvas. Affected UI users only; API users were unaffected.

### Changed
- Converted `5028 easy textSwitch.input` from a widget to a real input socket.
- Added `5036 PrimitiveInt` "Prompt Source (1=Gemini, 2=Manual)" wired into that socket. `PrimitiveInt` enforces an INT output regardless of how its own widget is serialized, so the toggle can't drift back to the broken state on future saves.
- Both `LipDub-Gemini-UI.json` and `Lipdub-Gemini-API.json` updated.

### Migration
If you previously edited the toggle by changing `5028`'s `input` widget directly, do it on the new `5036 PrimitiveInt` node instead. The textSwitch no longer has an editable widget.

## [1.5.0] — 2026‑05‑13

### Added
- **Windowed dubbing mode** — replace only a time range of audio while preserving the rest. Useful when you want to keep the original speaker's voice, mic effects, room tone, and ambient audio everywhere except inside the regenerated segment.
- New nodes inside the **Stage 1** group:
  - `5030 PrimitiveInt` "Dub Mode" — `0 = full dub` (default, prior behavior) / `1 = windowed dub`.
  - `5031 LTXVConcatAVLatent` — windowed concat feeding the source audio (encoded) into the av‑latent.
  - `5032 LTXVSetAudioVideoMaskByTime` — applies the time mask. Default widgets: `start_time=0.0`, `end_time=8.0`, `video_fps` wired from the source video, `mask_video=False`, `mask_audio=True`, `mask_init_value_video=1.0`, `mask_init_value_audio=0.0`, `slope_len=10`.
  - `5033 / 5034 easy conditioningIndexSwitch` — switches CFGGuider positive/negative between the full‑dub path (RefTokens `5006`) and the windowed path (SetMaskByTime `5032`).
  - `5035 easy anythingIndexSwitch` — switches Sampler1's `latent_image` between full‑dub `ConcatAV (4528)` and windowed `SetMaskByTime.av_latent`.
- Lazy switches: the unused branch is skipped at runtime, so full‑dub runs are exactly as fast as before.

### Usage
- **Full dub (default)**: `Dub Mode = 0`. Behaves like v1.4 — entire clip's audio regenerated.
- **Windowed dub**: `Dub Mode = 1`. Edit the `Audio Mask by Time` node's `start_time` and `end_time` to bracket the segment to regenerate. Audio outside that window — including mic effects, echo, and ambient — plays back unchanged (model‑preserved from the source latent). Video is still fully regenerated for lip‑sync.

### Limitations
- The IC‑LoRA emits clean studio speech inside the masked window. Effects like reverb tails will only bleed across the boundary via `slope_len` (the crossfade); they will not be perfectly reproduced *within* the regenerated segment.
- The LipDub IC‑LoRA was trained on full‑audio replacement, not time‑masked dubbing. The windowed mode is an opportunistic application of `LTXVSetAudioVideoMaskByTime` — quality is good in practice but not guaranteed for every clip.

## [1.4.0] — 2026‑05‑13

### Fixed
- **Output fps now matches the source video's fps.** Previously the workflow ran everything off a hard‑coded `PrimitiveFloat fps = 24`, so a 30 fps source produced a 24 fps output with the wrong duration. The hard‑coded primitive (`4989`) is gone.

### Changed
- New `VHS_VideoInfoLoaded` node (id `5029`) reads from `VHS_LoadVideo.video_info` (output 3) and provides the loaded fps as a FLOAT.
- All four fps consumers were rewired to read from it:
  - `LTXVConditioning.frame_rate` (generation conditioning)
  - `VHS_VideoCombine.frame_rate` (output encoding)
  - `LTXFloatToInt.a` → `LTXVEmptyLatentAudio.frame_rate` (audio latent)
  - `CreateVideo (Gemini Input).fps` (the MP4 sent to Gemini)
- Both `LipDub-Gemini-UI.json` and `Lipdub-Gemini-API.json` updated.

### Note
The output fps will match the source's *loaded* fps — i.e. whatever VHS actually read. If you use VHS_LoadVideo's `force_rate` widget (left at `0` by default = native), the output equals the source fps exactly. If you ever set `force_rate` to something else, that becomes the truth and the output follows.

## [1.3.0] — 2026‑05‑13

### Added
- **Gemini ↔ manual prompt toggle.** You no longer have to mute/rewire the Gemini node to use a hand‑written prompt.
  - New `PrimitiveStringMultiline` "Manual Prompt" node (id `5027`) — pre‑filled with an example, edit to taste.
  - New `easy textSwitch` node (id `5028`) titled *Prompt Source (1=Gemini, 2=Manual)*. Its `input` widget picks the source:
    - `1` → Gemini's output drives the positive CLIP encode and the debug Show Text.
    - `2` → Manual Prompt drives them instead; Gemini still runs but its output is ignored.
  - Wiring change: `5019 GeminiNode` → ~~`2483 CLIPTextEncode (Positive).text`~~ / ~~`5024 ShowText.text`~~ is now routed through `5028 textSwitch`, with `5027 PrimitiveStringMultiline` as the alternate input.
- Both `LipDub-Gemini-UI.json` and `Lipdub-Gemini-API.json` updated; UI groups preserved.

### Tip
To skip Gemini entirely (zero API calls), set the switch to `2` **and** right‑click the Gemini node → **Mute**. To re‑enable Gemini, unmute and set the switch back to `1`.

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
