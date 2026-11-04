# v1.0.1 — Add workflow screenshot

Documentation-only update: both READMEs now open with a screenshot of the SelfLift
Progressive Sampler (MiniMax H3) node in a ComfyUI workflow. No code changes.

---

# v1.0.0 — Initial public release

Progressive-resolution sampling for ComfyUI. Run the early denoising steps at low
resolution, lift the result to full resolution, and finish there — faster generation,
no training required.

## What's in this release

**SelfLift Progressive Sampler (Image, Euler/LCM)** — the paper's SelfLift-zero for
rectified-flow image backbones. Works with standard `euler` and `lcm` samplers, adds no
denoiser evaluations (an N-step schedule stays exactly N NFEs), and supports a separate
high-resolution model via `model_hires`.

**SelfLift Progressive Sampler (MiniMax H3)** — experimental audio-video adaptation.
Two modes: an external learned latent upscaler (practical default) or true SelfLift-zero
with the pixel-VAE anchor (`rho>0`). Includes optional high-resolution tiling to save VRAM.

**H3 Temporal State Transport (TST)** — a `MODEL` → `MODEL` patch node that improves
temporal stability of H3 video at inference time. No training, no extra model, ~1–2%
runtime overhead. Works with any standard sampler node, not only the SelfLift sampler.

## Status

**Research prototype.** The image sampler follows the
[SelfLift paper](https://arxiv.org/abs/2609.02036) closely and is the most mature part.
The H3 audio-video path and the TST node are experimental, lightly validated, and depend
on ComfyUI internals that can change between releases. The H3 adaptation is not validated
by the paper. Expect rough edges — please open an issue before reporting a bug.

## Installation

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/henry19801109/comfyui-SelfLift-with-lcm
```

Restart ComfyUI. The nodes appear under the `selflift` category. `psutil` is the only
extra dependency, needed solely by the optional memory diagnostics.

See [README.md](https://github.com/henry19801109/comfyui-SelfLift-with-lcm#readme) for
full parameter tables, diagnostics, and limitations.

## License

MIT © 2026 HERVINAI
