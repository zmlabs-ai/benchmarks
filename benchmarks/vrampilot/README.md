# VRAMPilot — benchmarks

> Fit any model to your GPU; recover from out-of-memory at runtime instead of crashing. Live: https://vrampilot.com

## What it does
VRAMPilot is an automation layer over llama.cpp that **auto-fits a GGUF model to your GPU and recovers from a runtime out-of-memory instead of crashing**. Given an impossible config it backs off **multi-axis** — quantizing the KV cache *first* to keep your context, only shrinking context when that is not enough — then keeps serving. It also fetches a verified llama-server binary on first run, persists the config that survived so the next boot starts there, and watches VRAM during inference to pre-empt a crash. Every figure below is end-to-end on a real GPU, no mocks.

## Measured benchmarks
| Claim | Measured value | Conditions | Source file |
|---|---|---|---|
| OOM-recovery, end-to-end (NVIDIA) | a forced `-c 262144` **recovers to `-c 131072`** (KV f16 → q8_0 → q4_0 first, then context) and **serves a real reply** — 4× more context than a context-only back-off | RunPod **RTX 3070 8 GB**, llama.cpp CUDA, Qwen2.5-7B-Instruct Q4_K_M (4.68 GB), batch-1, 3 e2e tests, no mocks | [`RESULTS.md`](RESULTS.md) |
| OOM-recovery on a second vendor (AMD) | a forced `-c 1000000` **recovers to `-c 500000`** on a real **Vulkan** OOM (not a CUDA error) and serves | Radeon 780M iGPU (Vulkan, b9550), **local single machine — disclosed**, Qwen2.5-1.5B Q4_K_M | [`AMD.md`](AMD.md) |
| Cross-vendor reach | **run + serve on NVIDIA, AMD and Apple M1**; clean sane-OOM recovery demonstrated on **NVIDIA + AMD**; the Apple/Metal mechanism is in place but a sane-OOM demo is **"not yet"** on a 7 GB CI runner — disclosed | RTX 3070 (RunPod) / Radeon 780M (local) / M1 (GitHub Actions `macos-14`) | [`CROSS_PLATFORM.md`](CROSS_PLATFORM.md) |
| Decode energy | **0.84 J/token** (CPU package, RAPL — **CPU-package only, disclosed**); boot 2539.42 J; GPU avg 37.4 W | Ryzen 7 8845HS + RTX 4070 Laptop 8 GB, EnergiBridge/RAPL, Qwen1.5-MoE 9.5 GB, ctx 8192, 256-token decode | [`ENERGY.md`](ENERGY.md) |
| In-inference watchdog | at **102 MiB < 400 MiB** floor → controlled restart, **recovered in 223.9 s** at a degraded config (ctx 8192 → 4096) **under sustained VRAM pressure**, then served | RTX 4070 Laptop 8 GB (nvidia-smi measured), llama-server b9437 CUDA, Qwen1.5-MoE-A2.7B Q4_K_M (9.5 GB on 8 GB) | [`WATCHDOG.md`](WATCHDOG.md) |
| Persistence (learns the working config) | cold **222.63 s** vs warm **223.7 s** — when boot lands first-try the cache saves no wall-clock; its value is **skipping failed attempts** (~220 s each) and starting at the survivor config | RTX 4070 Laptop 8 GB CUDA, llama-server 9437, Qwen1.5-MoE 9.5 GB, ctx 8192 & 32768 | [`PERSISTENCE.md`](PERSISTENCE.md) |
| Zero-prerequisite first run | download → SHA-256 → boot → serve in **7.6 s** (scrubbed host); a **clean Windows 11 VM with no GPU** (CPU fallback) validated end-to-end in **12.2 s** | pinned llama.cpp b9592, Qwen2.5-1.5B ctx 2048 | [`INSTALL.md`](INSTALL.md) |

## What these numbers do NOT prove
- **AMD recovery is a single founder machine** (Radeon 780M iGPU); not yet reproduced across AMD dGPU / ROCm.
- **Apple/Metal forced-OOM recovery is "not yet"** demonstrated — the mechanism boots, but a *sane* OOM needs more than a 7 GB CI runner. Run + serve on Apple **is** proven.
- **RAPL energy is CPU-package only** on this machine (no DRAM domain exposed) — it is not whole-system energy.
- **The persistence wall-clock win is conditional**: it appears when an attempt *fails* (each failed attempt costs a full ~220 s model load); when the planner lands first-try, cold ≈ warm.
- The watchdog's only runtime action is a **controlled restart** (llama.cpp has no hot KV-resize as of b9437 / b9592); a stream in flight at restart time is lost — stated in the file.

## Reproduce
`python runpod_ap.py` builds llama.cpp CUDA, downloads the GGUF, and runs the 3 end-to-end tests; locally, `python -m vrampilot.cli model.gguf` with a `llama-server` on PATH. Per-file conditions: [`../../AUDIT_SUMMARY.md`](../../AUDIT_SUMMARY.md).
