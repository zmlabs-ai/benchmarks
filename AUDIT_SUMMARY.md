# AUDIT_SUMMARY — nature of proof & honest caveats

> Transparency companion to [`benchmarks/`](benchmarks/). For every included file: what it measures, under which conditions, and **what it does not prove**. All entries are `MEASURED` (raw run with complete conditions). Numbers that were only modelled, synthetic, single-run-on-a-fragile-edge, or not reproducible were **excluded** — listed at the bottom with the reason.
>
> Paths to the original machines/users have been redacted. No secrets are present in this tree.

---

## memown — [`benchmarks/memown/`](benchmarks/memown/)

| File | Measures | Key conditions | Does NOT prove |
|---|---|---|---|
| `local_120b_8gb_decode.json` | **gpt-oss-120B decodes 1.50 tok/s on the 8 GB laptop** (n=1) | gpt-oss-120b MXFP4 ~63.4 GB, RTX 4070 Laptop **8 GB** + 16 GB RAM, `--cpu-moe -ngl 99 -c 4096`, NVMe expert-streaming, batch-1; prefill 122 tok @1.12 tok/s, decode 300 tok | speed (it is **capacity** — NVMe-bound ~1.5 tok/s); a multi-run median (this is n=1); the ≥10 tok/s target |
| `runpod_64gb_validation.json` | 7.62 tok/s decode | gpt-oss-120B MXFP4 63.4 GB, RunPod L40S 48 GB + 188 GB RAM, `--cpu-moe`, 3 stable runs | a laptop result (datacenter GPU); the ≥10 tok/s target is **explicitly not reached** |
| `e1_baseline.json` + `E1_BASELINE.md` | ~22 tok/s (best) / **~18–21 robust** | Qwen3-30B-A3B Q4_K_M, RTX 4070 Laptop 8 GB, batch-1, ncmoe sweep, N≥37 | the 22 figure is a **memory-fragile edge** (315 MiB free, OOM risk); the robust headline is 18–21 |
| `bench_results.json` | VRAM ~12.5 vs NVMe ~1–3.5 GB/s | RTX 4070, PCIe Gen4 x8, warm-up=3, N=30–50, 95 % CI | a model-level throughput (it's a memory-tier bandwidth measurement) |
| `wikitext_perplexity.txt` | quality cost: perplexity **+20 %** | GPT-2 124M, WikiText-2-raw, 561 windows / 287 644 tokens | that the method is **lossless** (it is **not**); GPT-2 is a toy reference |
| `capacity_proof.json` | survives OOM, zstd ~3.05× | synthetic logs, cgroup 1024 MB, lossless checksum oracle | a speed gain — this is **capacity** (survival), not tokens/second |

**Capacity headline ("120B on 8 GB"):** memown boots, serves **and decodes** a 120B-class model on an 8 GB GPU + system RAM at **~1.5 tok/s** (measured, n=1 — `local_120b_8gb_decode.json`). This is a **capacity** result, never a speed claim; with abundant RAM the same model reaches 7.62 tok/s on a datacenter GPU (`runpod_64gb_validation.json`).

---

## diciz — [`benchmarks/diciz/`](benchmarks/diciz/)

| File | Measures | Key conditions | Does NOT prove |
|---|---|---|---|
| `README.md` | 31.2 tok/s (7B) / 94.2 tok/s (0.5B) | Qwen2.5 Q4_K_M / q4f16, RTX 4070 Laptop 8 GB, Chrome WebGPU, greedy 128 tok | a median — these are **single measurements** (n=1), disclosed as such |
| `rife_native_vs_browser_2026-06-28.json` | RIFE-Nano ~4.13 ms/frame native (N=50) | RTX 4070, 480×270, fp32 both sides, byte-identical input, warm-up K=10 discarded, median + p95 | the native-vs-browser **ratio** is **not** headlined — the file's own `decision` field forbids it (outputs not bit-identical, 8.1 % divergence) |

---

## VRAMPilot — [`benchmarks/vrampilot/`](benchmarks/vrampilot/)

*The most disciplined set: each figure ships with its source run, and the build breaks if a published number diverges from its proof.*

| File | Measures | Key conditions | Does NOT prove |
|---|---|---|---|
| `RESULTS.md` | OOM-recovery 262144 → 131072 ctx, keeps serving | RunPod RTX 3070 8 GB, llama.cpp CUDA, Qwen2.5-7B-Q4_K_M, 3 e2e tests, no mocks | recovery on AMD ROCm / Apple Silicon (see below) |
| `ENERGY.md` | boot 2539 J, decode **0.84 J/token** | Ryzen 7 8845HS + RTX 4070 8 GB, EnergiBridge/RAPL, Qwen1.5-MoE | DRAM energy — **RAPL = CPU package only** (disclosed) |
| `WATCHDOG.md` / `PERSISTENCE.md` / `INSTALL.md` | watchdog 223.9 s @102 MiB, cold persist 222.6 s, install 7.6 s | logged, dated, reproducible | — |
| `CROSS_PLATFORM.md` | runs/serves across CUDA, Vulkan, Metal | per-platform logs | **Apple/Metal forced-OOM recovery "not yet demonstrated"** (stated) |
| `AMD.md` | AMD recovery (iGPU 780M Vulkan, 1M → 500K) | single founder machine, run unique, 1.5B model | reproducibility across AMD dGPU/ROCm (single-machine — disclosed) |

---

## GlassBreakr — [`benchmarks/glassbreakr/`](benchmarks/glassbreakr/)

| File | Measures | Key conditions | Does NOT prove |
|---|---|---|---|
| `framegen_8x_rtx4070_2026-06-24.json` | **8× = output frames** (7 interpolated in 15.8 ms @1080p) | RTX 4070 Laptop, RIFE-Nano-S, warm-up 6, median of 30, blocking readback | image **quality** (no PSNR/SSIM at 8×); render speed (game render is unchanged); real responsiveness is ~2× |
| `8X_PROOF.md` / `BENCH.md` | frame-gen throughput per tier; dGPU impact 0–9.6 % | HP OMEN 8845HS + RTX 4070 + 780M, cross-checked | a high-end desktop-GPU equivalence (the file marks it **not published**); a **zero** GPU impact (impact is ≤9.6 %, never zero) |

---

## Deliberately EXCLUDED (and why)
These exist in the founder's archives but are **kept out** of this public benchmark set:

- **memown `cmr_ttft_120b.json` (×52.9 TTFT)** — measured but **n=1, single 179-token prompt**; the ratio depends on prompt length and was self-classified "not verified". → re-measure (N≥50, several prompt lengths) before publishing.
- **memown `runpod_8gb_reverify.json` (22.16 tok/s on 120B)** — host had **251 GB server RAM**, self-labelled "optimistic"; misleading for an 8 GB laptop.
- **memown `benchmark_final.txt` (7.8×) / `rtx4070_benchmark.txt` (614×)** — **synthetic tensors, no real LLM**; the implied "speed" is not an end-to-end product gain.
- **Frame-gen ratios 73× / 42× / 41×** — not bit-identical outputs (up to 8.1 % divergence); the underlying file forbids headlining the ratio.
- **Marketing-only figures** ("= a high-end desktop GPU", "119 FPS", "100× less money", "0.16 ms/frame") — modelled/estimated or submit-time artifacts, not measured product results.

*Created during an internal honesty audit; every figure is checked against its proof file before publication.*
