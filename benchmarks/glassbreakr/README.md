# GlassBreakr — benchmarks

> More FPS through frame generation, using your PC's idle graphics chip. Live: https://glassbreakr.com

## What it does
GlassBreakr is a Windows frame-generation overlay (RIFE-Nano). It inserts AI-interpolated frames between real ones — the **same kind of technique as NVIDIA DLSS Frame Generation and Lossless Scaling** — and can run that interpolation on your **integrated GPU** while the dedicated GPU keeps rendering the game. The headline **"up to 8×"** is a count of **displayed / output frames** (1 captured + 7 generated), **not** a render-speed or image-quality claim. The frame-generation nature is disclosed alongside every figure below.

## Measured benchmarks
| Claim | Measured value | Conditions | Source file |
|---|---|---|---|
| "Up to 8×" = output frame throughput | **8× = 1 captured + 7 AI-interpolated frames**; the 7 are generated in **15.8 ms** median (min 14.19 / max 17.82) at 1080p → sustains 8× output up to **~63.3 captured fps** (e.g. 30 fps render → ~240 fps output) | RTX 4070 Laptop, onnxruntime 1.24.4 DirectML, `rife_nano_s_flow_320x192.onnx`, warmup 6, median of 30, blocking readback | [`framegen_8x_rtx4070_2026-06-24.json`](framegen_8x_rtx4070_2026-06-24.json) · [`8X_PROOF.md`](8X_PROOF.md) |
| Tier multipliers (generated) | **FREE 4× / TURBO 6× / ULTRA 8×**, verified two ways: shipped-binary disassembly (`{4, 6, 8}`; `compute_timesteps` emits N−1 distinct frames) and a Cyberpunk 2077 beta test | shipped binary + beta-tester capture | [`BENCH.md`](BENCH.md) · [`8X_PROOF.md`](8X_PROOF.md) |
| Real (responsive) frames | **~2×** — Subway Surfers ~30 → **63 fps** output, stable over **11,500+ frames**; Unity POOLS 59.5 → 68.5 fps real | RTX 4070 Laptop, RIFE; POOLS = Unity AAA scene, 1080p, 20 s | [`BENCH.md`](BENCH.md) · [`8X_PROOF.md`](8X_PROOF.md) |
| Impact on the main (dedicated) GPU | **9.6 %** (never zero); the AI interpolation can run on the iGPU with measured dedicated-GPU AI load **0 %** in heterogeneous mode | HP OMEN 8845HS + RTX 4070 Laptop + Radeon 780M, `benchmark_pools.json` | [`BENCH.md`](BENCH.md) |

## What these numbers do NOT prove
- **8× is not image quality.** The model is a small RIFE-Nano; **no PSNR/SSIM at 8× was measured**. It is a generated-frame count, not a blind-tested quality grade.
- **8× is not render speed or responsiveness.** The game's own render rate is unchanged; input latency rises ~15–25 ms. Real responsiveness is **~2×**.
- **What you *see* is capped by your monitor** — e.g. 256 generated frames show as ~180 on a 180 Hz panel (≈ 4× visible).
- **dGPU impact is low but never zero** (≤ 9.6 %).
- No high-end-desktop-GPU equivalence is claimed (that estimate was removed from the public set).

## Reproduce
Frame-gen throughput: `measure_ultra_8x.py` (warmup 6, median of 30) reproduces the 15.8 ms / 8× figure on an RTX 4070-class GPU with onnxruntime DirectML. The tier multipliers are verifiable from the shipped binary's disassembly. Per-file conditions and the deliberately excluded ratios: [`../../AUDIT_SUMMARY.md`](../../AUDIT_SUMMARY.md).
