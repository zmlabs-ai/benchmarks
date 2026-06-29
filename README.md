# ZMLabs — Benchmarks & Proofs

> **Honesty first.** If we can't measure it, we don't sell it. Every headline number in this repository is backed by a raw measurement file under [`benchmarks/`](benchmarks/), with full conditions (model, quantization, hardware, batch, sample count, warm/cold). Capacity is never reported as speed. Estimated or single-run numbers are labelled as such — or kept out entirely. **"Stand behind" means measured, not believed**: a number ships only with its proof file attached, and anything we measured but *could not* stand behind — fragile edges, non-reproducible ratios — is listed with its reason in [`AUDIT_SUMMARY.md`](AUDIT_SUMMARY.md).
>
> ZMLabs — Sète, France · contact: contact.zmlabs@proton.me

This repository gathers the **measured benchmarks** behind ZMLabs' four products. It contains only proofs with complete measurement conditions; modelled, synthetic, or not-yet-reproduced numbers are deliberately excluded (see [`benchmarks/BENCHMARKS_MANIFEST.md`](benchmarks/BENCHMARKS_MANIFEST.md) and [`AUDIT_SUMMARY.md`](AUDIT_SUMMARY.md)).

---

## The four products

### 🧠 memown — run a large local AI on a small machine
A local inference layer focused on **capacity, not raw speed**: it boots, serves **and decodes** a **120B-class MoE model on an 8 GB GPU + system RAM** — at **~1.5 tok/s** (n=1, NVMe-bound; see [`benchmarks/memown/local_120b_8gb_decode.json`](benchmarks/memown/local_120b_8gb_decode.json)). This is a capacity result, explicitly **not** a speed claim. On a mid-range MoE (Qwen3-30B-A3B Q4_K_M, RTX 4070 8 GB) it serves at **~18–21 tok/s robustly** (up to ~22 at a memory-fragile edge — disclosed). Quality changes from memory tiering are reported honestly (e.g. perplexity +20 % on a GPT-2 / WikiText-2 reference — **not** "lossless").
→ proofs: [`benchmarks/memown/`](benchmarks/memown/)

### 🎮 diciz — an LLM that runs in your browser
A WebGPU runtime that runs real GGUF models **100 % on-device, no install, no Python**. Measured throughput (single measurement, disclosed): **31.2 tok/s** (Qwen2.5-7B Q4_K_M) and **94.2 tok/s** (Qwen2.5-0.5B) on an RTX 4070 Laptop via Chrome WebGPU. Also ships browser-side frame interpolation (RIFE-Nano), measured rigorously (N=50, warm-up discarded).
→ proofs: [`benchmarks/diciz/`](benchmarks/diciz/)

### 🛟 VRAMPilot — never crash on out-of-memory again
A UX/automation layer over llama.cpp that **auto-fits a GGUF model to your GPU and recovers from a runtime out-of-memory instead of crashing**. Measured end-to-end (no mocks): a forced OOM at **262 144 ctx backs off to 131 072 ctx and keeps serving** (RTX 3070 8 GB, Qwen2.5-7B-Q4_K_M); decode energy **0.84 J/token** (RAPL, CPU-package only — disclosed). Cross-platform support is reported precisely: CUDA & Vulkan demonstrated; Apple/Metal recovery **not yet demonstrated** (stated as such).
→ proofs: [`benchmarks/vrampilot/`](benchmarks/vrampilot/)

### 🖼️ GlassBreakr — more frames per second through frame generation
A Windows frame-generation overlay (RIFE-Nano). The **"up to 8×"** figure is a **throughput of generated frames** (7 interpolated frames produced in 15.8 ms at 1080p, RTX 4070), disclosed as frame-gen — **not** an image-quality claim and **not** a render-speed claim. Real responsiveness is ~2×; impact on the main GPU is **≤ 9.6 %** (measured, disclosed as low — never "zero").
→ proofs: [`benchmarks/glassbreakr/`](benchmarks/glassbreakr/)

---

## How to read these benchmarks
- **Capacity ≠ speed.** "Runs a 120B model on 8 GB" is about *fitting and serving* — even on a **16 GB-RAM** laptop. It does decode (~1.5 tok/s, n=1, measured), but that is reported as a **capacity** result, never as a speed number.
- **Throughput ≠ quality.** A frame-gen "8×" is frames produced, not visual fidelity.
- **Conditions matter.** Every number lives next to its model, quantization, hardware, sample count and warm/cold state. A bare number is not a benchmark.
- **Single runs are labelled.** Where n=1, it is said so; rigorous figures carry N and discarded warm-up.

See [`AUDIT_SUMMARY.md`](AUDIT_SUMMARY.md) for the per-file nature-of-proof and the explicit caveats (what each measurement does **not** prove).

*Published by ZMLabs. Every figure here is a real measurement with its proof file attached.*
