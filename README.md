# Lipdub‑Gemini Workflow

A ComfyUI workflow for **LTX 2.3 LipDub IC‑LoRA** with **Google Gemini** acting as an automatic transcriber + translator + prompt enhancer.

Drop a source video in, pick a target language, and the workflow:

1. Transcribes the source audio and translates the dialogue with Gemini.
2. Writes an LTX‑formatted prompt describing the speaker, language, and dialogue.
3. Generates a lip‑synced replacement at 1920×1088 with the original speaker's face and the new dialogue.
4. Auto‑adapts to whatever source video length you give it (no manual frame‑count tweaking).

## Files

- **`LipDub-Gemini-UI.json`** — the workflow in ComfyUI **UI format** (with positions, links, and 7 colored groups: *Load Models*, *Load Original Video*, *Stage 1*, *Gemini*, *Stage 2*, *Decode*, *Output*). **This is the one you want to load in the ComfyUI canvas.**
- `Lipdub-Gemini-API.json` — the same workflow in ComfyUI **API format** (logic only, no visual data). Use this one when calling ComfyUI via the API.
- `CHANGELOG.md` — what was changed relative to the original workflow and why.

## Requirements

### ComfyUI

A current build of ComfyUI with the built‑in **API nodes** enabled (provides the `GeminiNode`). You'll need a Google AI Studio / Vertex API key configured in ComfyUI's API node settings.

### Custom node packs

| Pack | Used for |
|---|---|
| [`ComfyUI-VideoHelperSuite`](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | `VHS_LoadVideo`, `VHS_VideoCombine` |
| [`ComfyUI-LTXVideo`](https://github.com/Lightricks/ComfyUI-LTXVideo) | LTX checkpoint loaders, samplers, IC‑LoRA, tiled VAE decode |
| [`ComfyUI-KJNodes`](https://github.com/kijai/ComfyUI-KJNodes) | `ImageResizeKJv2`, `ResizeImageMaskNode` |
| [`ComfyUI-Easy-Use`](https://github.com/yolain/ComfyUI-Easy-Use) | `easy simpleMath` (LTX‑valid length snapper) |
| [`ComfyUI-Custom-Scripts`](https://github.com/pythongosssss/ComfyUI-Custom-Scripts) | `ShowText\|pysssss` (display Gemini output) |

Install via [ComfyUI‑Manager](https://github.com/Comfy-Org/ComfyUI-Manager) and let it pull missing nodes after loading the workflow.

### Models (place in `ComfyUI/models/...`)

| File | Path | Notes |
|---|---|---|
| `ltx-2.3-22b-dev.safetensors` | `checkpoints/` | Main LTX 2.3 22B model |
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` | `loras/` | Distilled LoRA (faster sampling) |
| `ltx-2.3-22b-ic-lora-lipdub-0.9.safetensors` | `loras/ltxv/ltx2/` | LipDub IC‑LoRA |
| `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` | `upscale_models/` | Latent ×2 upscaler |
| `comfy_gemma_3_12B_it.safetensors` | `text_encoders/` | LTX 2.3 text encoder |

## Usage

1. Load `LipDub-Gemini-UI.json` in ComfyUI (**Workflow → Load**, or drag it onto the canvas). You'll see 7 colored groups laid out left‑to‑right.
2. In the **Load Original Video** group, choose your source clip in `Load Video (Upload)`.
3. In the **Gemini** group, set the `Google Gemini` node's `prompt` widget to the **target language** (e.g. `Spanish`, `French`, `Hebrew`, `Russian`).
4. Hit **Run**.

The workflow will:
- Load the source video (no manual frame‑cap needed).
- Snap the LTX latent length to the largest valid `N×8+1` ≤ source length.
- Send Gemini a compressed MP4 of the source so it can hear/see the dialogue.
- Use Gemini's response as the LTX positive prompt.
- Generate the lip‑synced output to `ComfyUI/output/output_*.mp4`.

## Workflow groups

The UI workflow ships with 7 colored groups laid out left→right:

| Group | What it does |
|---|---|
| **Load Models** | Checkpoint, distilled LoRA, IC‑LoRA, audio VAE, latent upscaler, text encoder |
| **Load Original Video** | VHS Load Video + auto length snapping (`floor((n‑1)/8)*8+1`) |
| **Gemini** | System prompt, downscaled video packing, Google Gemini call, debug Show Text |
| **Stage 1** | 960×544 IC‑LoRA‑conditioned base generation (8 denoising steps) |
| **Stage 2** | Latent ×2 upscale + refinement pass at 1920×1088 (5 denoising steps) |
| **Decode** | Tiled VAE decode + audio decode + crop guides |
| **Output** | `VHS_VideoCombine` H.264 MP4 encode (CRF 17, yuv420p, bt709) |

The API JSON (`Lipdub-Gemini-API.json`) preserves the same logic but stores no visual data, so when loaded fresh it'll auto‑lay out without the colored rectangles. Every node's `_meta.title` is prefixed with a `[Group]` tag in both files so the structure stays readable either way.

## Tuning knobs

| What | Where | Default | Try |
|---|---|---|---|
| Output quality | **Output** → `Video Combine` → `crf` | `17` | `15`‑`19` (lower = sharper, bigger file) |
| Detail refinement | **Stage 2** → `ManualSigmas` → `sigmas` | 5 steps | Add more sigmas for more detail (slower) |
| Distilled LoRA strength | **Load Models** → `Load LoRA` → `strength_model` | `0.5` | `0.3`‑`0.4` for sharper, needs more steps |
| Tile overlap | **Decode** → `LTXV Tiled VAE Decode` → `overlap` | `8` | Max is 8 (latent units, ≈ 256 px) |
| Gemini context resolution | **Gemini** → `Resize Image v2` → `width/height` | `360×360` | Bump to `512×512` for better lip reading |

## Notes

- Output is 1920×1088 (16:9.07). LTX 2.3's VAE compresses 32× spatially, which forces dimensions to multiples of 32 — `1088 ≠ 1080`.
- A 30 s + source video will require significant VRAM. If you OOM, trim the source externally or use the Load Video node's `skip_first_frames` widget.
- The IC‑LoRA only handles **a single speaker**. Multiple visible faces will all get lip‑synced.
