# comfyui-SelfLift-with-lcm

[中文说明](README_CN.md) · [License](LICENSE)

![SelfLift Progressive Sampler (MiniMax H3) in a ComfyUI workflow](docs/screenshot-h3-workflow.png)

Progressive-resolution sampling for ComfyUI: run the early denoising steps at low resolution, lift the result to full resolution, and finish there — faster generation with no training.

Based on the [SelfLift paper](https://arxiv.org/abs/2609.02036) (SelfLift-zero) for rectified-flow image models, plus an experimental MiniMax H3 audio-video adaptation that the paper does not validate, and an experimental H3 port of [TST](https://arxiv.org/abs/2609.08505) temporal-attention correction.

> **Status: research prototype.** The image sampler follows the paper closely and is the most mature part. The H3 audio-video path and the TST node are experimental, lightly validated, and depend on ComfyUI internals that can change between releases. Expect rough edges and please open an issue before reporting a bug.

---

## Installation

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/henry19801109/comfyui-SelfLift-with-lcm
```

Restart ComfyUI. The nodes appear under the `selflift` category.

`psutil` is the only extra dependency and is used solely by the optional memory diagnostics (`SELFLIFT_MEMORY_LOG=1`):

```bash
pip install psutil
```

### Requirements

- A ComfyUI build with `comfy.samplers`, `comfy.model_sampling`, and `comfy.patcher_extension` (any recent nightly).
- For the H3 sampler and the TST node: a ComfyUI build with MiniMax H3 support (`comfy.ldm.minimax`). Without it the H3 nodes raise a clear error and the rest of the plugin still works.
- For the external H3 upscaler (optional): a checkpoint from [LBH-123-AI/Minimax_h3_latent_Upscaler](https://huggingface.co/LBH-123-AI/Minimax_h3_latent_Upscaler) placed in `ComfyUI/models/latent_upscale_models/`.

## Nodes

| Node | Type | What it does |
| --- | --- | --- |
| **SelfLift Progressive Sampler (Image, Euler/LCM)** | `MODEL` + `LATENT` → `LATENT` | The paper's SelfLift-zero for rectified-flow image backbones, with Euler and LCM sampler support |
| **SelfLift Progressive Sampler (MiniMax H3)** | `MODEL` + `LATENT` → `LATENT` | Experimental audio-video adaptation, plus optional learned latent upscaler and high-resolution tiling |
| **H3 Temporal State Transport (TST)** | `MODEL` → `MODEL` | Training-free temporal-stability correction for H3 video, usable with any standard sampler node |

The sampler nodes take `sampler` and `sigmas` inputs like `SamplerCustom`. Connect the standard `euler` sampler from `KSamplerSelect` and the model's normal scheduler; other samplers are rejected by design.

---

## SelfLift Progressive Sampler (Image)

The paper's SelfLift-zero for rectified-flow image backbones. Connect a 4D image latent (e.g. *Empty Latent Image*) and the model's own VAE.

| Parameter | Default | Meaning |
| --- | --- | --- |
| `transition_step` | 0 | Steps run at low resolution. **0 = automatic**: keeps the paper's quarter of the steps at full resolution, which reproduces the paper's values exactly (6 of 8 for Z-Image-Turbo, 3 of 4 for FLUX.2-Klein) and turns a 6-step schedule into 4 low + 2 high. Any other value is used as-is |
| `lowres_scale` | 0.5 | Spatial scale of the low-res prefix |
| `rho` | 0.3 | Fraction of high-risk locations corrected toward the pixel-VAE anchor. FLUX.2-Klein: 0.4. 0 disables the anchor |
| `w_min` / `w_max` | 0.5 / 1.0 | Correction strength range inside the selected locations |
| `latent_upsample` | nearest | Direct-lift interpolation; `bilinear` is an option |
| `model_hires` (optional) | — | Separate model for the high-resolution stage (e.g. a different checkpoint or LoRA stack). Must share the same architecture and latent format; the low-res prefix always runs on `model` |

The transition adds no denoiser evaluations: an N-step schedule stays exactly N NFEs, plus one VAE decode → upscale → re-encode round trip unless `rho=0`.

### LCM sampling

Both nodes also accept the standard `lcm` sampler, which enables few-step distilled models (LCM, SDXL-Lightning, SD-Turbo). The two families differ in how a step advances:

- **Euler** integrates a derivative, so the discarded transition update is rebuilt from the low-resolution state.
- **LCM** replaces the state with the clean estimate and re-noises it at the next sigma, so the transition rebuilds that same replace-and-renoise step from the corrected endpoint.

The rebuild is expressed through the model's own `noise_scaling`, so the sigma family (flow vs. DDPM) is handled by the model rather than by this plugin. The `lcm` sampler therefore runs on any model, rectified-flow or DDPM.

Requirements:

- The sampler must be the standard **`lcm`** from `KSamplerSelect`.
- For distilled-model quality, patch the model with **`ModelSamplingDiscrete` → `lcm`** (the *ModelSamplingDiscrete (advanced)* node) before it reaches the sampler. That patch installs the LCM `c_skip`/`c_out` mixing and the distilled timestep schedule. It is a quality concern rather than a requirement of the sampler — SelfLift logs a warning when the sampler runs on an unpatched model.
- `cfg` should stay low (1.0–2.0); distilled models are trained without guidance.

A typical 6-step setup: `steps=6`, `sampler=lcm`, `transition_step=0` (auto → 4 low + 2 high), `cfg=1.0`, scheduler `simple` or `normal`.

To use it, connect `KSamplerSelect → lcm` into the `sampler` input. For a distilled model, wire the model through **ModelSamplingDiscrete (advanced)** with `sampling=lcm` first.

Note that the LCM path offsets the noise seed for the transition and the high-resolution phase, because LCM draws fresh noise at every step and the low-resolution prefix has already consumed its share of the sequence.

---

## SelfLift Progressive Sampler (MiniMax H3)

Experimental H3 audio-video adaptation (not paper-validated). Two modes:

- **External upscaler (default)**: an installed H3 checkpoint with `rho=0` — a learned latent-only lift. Practical default, but not SelfLift-zero.
- **SelfLift-zero**: `upscaler_model=none` with `rho>0`. Suggested H3 starting point: `rho=0.6`, `w_min=w_max=1` (see *Diagnosing H3*).

Extra inputs: `upscaler_model`, `upscaler_unload` (default on: unloads the upscaler from VRAM right after the lift, before the high-resolution stage) and `highres_tiling` (experimental: splits the high-res phase into 1–8 spatial tiles to save VRAM; only the first tile's audio is kept, there is no cross-tile attention, ControlNet is unsupported, and quality/speed may change).

### Optional H3 upscaler

Download from [LBH-123-AI/Minimax_h3_latent_Upscaler](https://huggingface.co/LBH-123-AI/Minimax_h3_latent_Upscaler) into `ComfyUI/models/latent_upscale_models/` and restart. The node selects the first filename containing `h3`, otherwise `none` (nearest-neighbor lift).

## H3 Temporal State Transport (TST)

A `MODEL` → `MODEL` patch node that improves the temporal stability of H3 videos at inference time — no training, no extra model, ~1–2% runtime overhead. Typical problems it counteracts: details that flicker or morph between frames (logos, on-screen text, textures), identity drift of people and outfits, and physically implausible motion. It works with any standard sampler node, not only the SelfLift sampler. It cannot create detail the model does not have.

| Parameter | Default | Meaning |
| --- | --- | --- |
| `tau` | 0.2 | Correction strength. 0.2 is the recommended value; 0.5 is visibly too strong. 0 disables correction but keeps diagnostics |
| `log_diagnostics` | on | Logs one `[H3 TST]` line per model forward |

How it works, in one paragraph: instead of blindly strengthening cross-frame attention, TST measures a signed "Spectral Tension" of the frame-level attention operator and corrects only imbalanced heads — sharpening over-mixed ones, softening fragmented ones — with stronger correction in deeper layers and earlier steps. Theory and derivations are in the [paper](https://arxiv.org/abs/2609.08505).

The diagnostics line reports latent frames, grid rows per frame, tension before/after the within-call correction, mean gamma, the share of corrected heads, and TST's own runtime. On real H3 runs the correction direction matched the exact attention operator for 89–100% of heads. Set `SELFLIFT_TST_EXACT=1` before starting ComfyUI to log the exact-operator comparison on one layer per forward (debug only, slow).

Limitations: MiniMax H3 models only (other models are skipped silently); **not compatible with `highres_tiling`** (skipped with a console warning).

## Diagnosing H3

Same prompt and seed, `transition_step=6`, `lowres_scale=0.5`:

| Test | `upscaler_model` | `rho` | Weights | Meaning |
| --- | --- | ---: | --- | --- |
| Direct route | `none` | 0 | any | Nearest-neighbor latent lift only |
| Pixel route | `none` | 1 | `1 / 1` | Pure H3 VAE pixel anchor |
| Paper-like SelfLift-zero | `none` | 0.3 | `0.5 / 1` | Paper's image parameters |
| Strong H3 SelfLift-zero | `none` | 0.6 | `1 / 1` | Tested H3 starting point |
| External lifter | H3 checkpoint | 0 | any | Learned H3 lift |

A controlled single-seed H3 run found: clean native baseline and clean pure pixel anchor; widespread artifacts on the nearest route that the paper's image setting (`rho=0.3`) mostly left visible; `rho=0.6` with `w_min=w_max=1` removed the main artifacts. Single-seed evidence only — H3's direct-lift error is broader than on the paper's image backbones, so start strong and reduce only if the result is overly smooth.

## Logging and environment variables

- `[SelfLift plan]` — resolved low/target latent shapes, NFE counts, transition sigmas, enabled lift routes
- `[SelfLift timing]` — wall-clock time per step and per stage (low-res, transition, high-res)
- `[SelfLift upscaler]` / `[SelfLift tiling memory]` — upscaler inference and tiling memory estimates (heuristics, not measured peaks)
- `[H3 TST]` — TST per-forward diagnostics
- `SELFLIFT_TIMING_SYNC=1` — CUDA-synchronized timing (slower; for diagnostics)
- `SELFLIFT_MEMORY_LOG=1` — host/device memory snapshots at stage boundaries
- `SELFLIFT_DEBUG=1` — dump transition intermediates as PNGs under `debug/` (slow, memory-hungry)
- `SELFLIFT_TST_EXACT=1` — TST exact-operator calibration probe (debug only)

## Notes and limitations

- Only the standard `euler` (with `s_churn=0`) and `lcm` samplers are accepted; other samplers are rejected. `euler` requires a rectified-flow model; `lcm` runs on any model.
- `noise_mask` is supported on both samplers with Set Latent Noise Mask semantics (1 = generate, 0 = keep the original content; use an initialized latent to have content worth keeping). Masks at any resolution are resized to the latent grid, and a time length of 1 is shared over all frames. The keep-region is pinned to the original latent at every step, and the artifact-aware correction is restricted to the generate region. `noise_mask` is not compatible with `highres_tiling`.
- Use the VAE belonging to the sampled model so the pixel anchor stays in the same latent space.
- H3's 768-pixel short edge becomes 384 px at `lowres_scale=0.5`, which may sit outside the backbone's training distribution — validate per model.
- SelfLift-rich (the distilled lifter + On-Policy Self Recovery) requires training and is not included.
- On the H3 path, a 2× spatial lift is roughly a 3.9× increase in attention tokens, so the high-resolution phase is not cheaper per step than the low-resolution one — the speedup comes from the low-res prefix, not from the lift itself.

## Contributing

Issues and pull requests are welcome. Please include the ComfyUI version, the model checkpoint, and the full console log (the `[SelfLift plan]` line is the most useful single line to quote).

If you change the sampler or the transition logic, please state which model and sampler you validated against — the Euler and LCM paths share very little beyond the entry point.

## References and acknowledgements

- SelfLift paper: [SelfLift: Accelerating Few-Step Diffusion via Self-Recovering Resolution Transition](https://arxiv.org/abs/2609.02036)
- TST paper and code: [Temporal State Transport in Video Generation](https://arxiv.org/abs/2609.08505), [lytang63/temporal-state-transport](https://github.com/lytang63/temporal-state-transport)
- Optional MiniMax H3 latent upscaler checkpoint and download: [LBH-123-AI/Minimax_h3_latent_Upscaler](https://huggingface.co/LBH-123-AI/Minimax_h3_latent_Upscaler)
- Original ComfyUI integration and inference implementation: [LBH-123-AI/Comfyui_Minimax_h3_latent_Upscaler](https://github.com/LBH-123-AI/Comfyui_Minimax_h3_latent_Upscaler)

Thanks to LBH-123-AI for publishing the MiniMax H3 latent upscaler weights and ComfyUI implementation. They made the optional learned H3 lifting path in this plugin possible. This external lifter remains separate from the SelfLift paper's SelfLift-rich model.

```
@article{wen2026selflift,
  title={SelfLift: Accelerating Few-Step Diffusion via Self-Recovering Resolution Transition},
  author={Wen, Tingyan et al.},
  journal={arXiv:2609.02036},
  year={2026}
}
```

## License

[MIT](LICENSE) © 2026 HERVINAI
