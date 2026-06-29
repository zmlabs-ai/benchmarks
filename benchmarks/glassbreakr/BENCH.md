# GlassBreakr — measured & verified figures (source of truth)

Every figure published on the GlassBreakr GEO site must appear **verbatim** in
this file, or `citable check` fails the build. Figures come from the product's
measured benchmarks (`zmlabs-glassbreakr/BENCHMARK_RESULTS.md`,
`benchmark_pools.json`), the shipped-binary disassembly, and the beta tester's
Cyberpunk 2077 test. All proofs (shipped binary, screenshots, beta report + video,
SHA-256) are in the repo `<org>/glassbreakr-proofs`.

Test machine for the benchmark runs unless stated: HP OMEN, Ryzen 7 8845HS + RTX
4070 Laptop GPU (dGPU) + Radeon 780M (iGPU).

## Frame multiplier — verified

GlassBreakr generates AI frames between real frames. Per tier the engine generates
up to: FREE **4x**, TURBO **6x**, ULTRA **8x**. This is verified two ways:
1. **Shipped binary** (March-5 build): the tier multipliers are {4, 6, 8} passed
   straight into the pipeline, with **no 4x clamp** bundled; `compute_timesteps(N)`
   emits N−1 distinct warped frames, each counted once. (Disassembly + the binary
   itself are in the proofs repo.)
2. **Beta tester test on Cyberpunk 2077** (below).

**Honest nuance — generated vs displayed:** a frame is *generated* by the engine,
but a monitor only *shows* its refresh rate. On a **180 Hz** screen the ULTRA
output appears as ~180 fps (≈ **4x** visible); on a faster display you see more. So
"up to 8x generated" is verified; what you *see* depends on the monitor.

## Cyberpunk 2077 — beta tester test

Captured live with GlassBreakr's own overlay over Cyberpunk 2077 (screenshots
verified first-hand; full PDF report + screen recording in the proofs repo, 2026-06-07):

- FREE: 22 → **120 fps**. (This single run reads ~5.45×, *above* FREE's 4x cap — a
  frame-counter artifact in that capture; FREE's sustained figure is 4x.)
- TURBO: 30 → **148 fps** (**4.9x**).
- ULTRA: 45 → **283 fps** (**6.3x**).

## Live tests (Subway Surfers, browser)

- RIFE interpolation, ~30 fps game → output **63 fps** (real + interpolated),
  inference avg **21.0ms**, RTX 4070 AI load **0%**, stable over **11,500+ frames**.
- RIFE + ESPCN 2x upscale, ~16 fps game → output **33 fps** at 960x540
  (AI-upscaled, not native), E2E avg 30.2ms, frame pacing 95-96%,
  stable over **19,700+ frames**.
- FlowUp pipeline, **720p native** output, **45-48 fps**, flow 9-11ms,
  warp **1.7-2.2ms**, E2E **16-18ms**, stable over **53,000+ frames**.

The Subway Surfers runs show ~**2x** output; the Cyberpunk tiers above show the
higher per-tier multipliers (up to 8x generated).

## iGPU inference benchmark (Radeon 780M, DirectML, 500 frames, no display)

- 270p (480x270): inference 12.87ms → **74.9 fps**.
- 360p (640x360): inference 25.18ms → 38.0 fps.
- ESPCN 2x upscale on iGPU DirectML: **3.49ms**.

## POOLS (Unity AAA, RTX 4070 Laptop, 1080p, 20s) — benchmark_pools.json

- Baseline: **59.5 fps** average (min 27.6).
- GlassBreakr hybrid: real output **68.5 fps** (295 interpolations). The "2x
  effective 119" value is a calculation (59.5 × 2), not delivered frames.
- dGPU impact: **9.6%**.

## Architecture fact

The AI interpolation runs on the integrated GPU (iGPU) while the dedicated GPU
keeps rendering the game; measured dedicated-GPU AI load in heterogeneous mode is
**0%**. "Output FPS" counts real frames plus AI-generated frames — the game's own
simulation is not made faster.

## Honest caveats

- Output FPS = real frames + generated frames; the game itself is not faster.
- "Generated" frames depend on the display to be *seen* (the 180 Hz nuance above).
- ULTRA 8x is verified as a generated frame count, **not** as a blind-tested quality
  grade (the model is a small RIFE-Nano; no PSNR/SSIM at 8x measured).
- 540p output is AI-upscaled, not native.
- Benchmark runs above are dominated by Subway Surfers + one Unity scene (POOLS);
  the Cyberpunk figures are the beta tester's test. Broader per-game measurement is ongoing.
