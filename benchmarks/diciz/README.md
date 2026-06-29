# DiciZ engine benchmarks

Reproducible decode-speed (and energy) benchmarks comparing on-device LLM engines, used to decide the
cross-platform engine strategy. All scripts drive a real headless Chrome (WebGPU) or a native binary and
measure **decode tokens/second** on the **same model + same prompt**.

## Results (measured 2026-06-10, evening — after the engine speed sprint)

**Machine:** Windows 11, NVIDIA GeForce RTX 4070 Laptop (8 GB) + AMD Radeon 780M iGPU, Chrome (WebGPU).
**Model:** Qwen2.5-0.5B-Instruct, ~4-bit (q4km GGUF / q4f16 for the ONNX-format engines). Greedy decode, 128 tokens.
Raw transcript: `<path>/cli-test/_diciz_speed_2026-06-10.out`.

| Engine | Approach | Device | decode t/s | TTFT (18-tok prompt) |
|---|---|---|---:|---:|
| llama.cpp **Vulkan** (pocketdc) | native, compiled | NVIDIA RTX 4070 | **362**¹ | — |
| llama.cpp **Vulkan** (pocketdc) | native, compiled | AMD 780M iGPU | **361**¹ | — |
| **WebLLM** (MLC / Apache-TVM) | compiled WebGPU, **f16 compute** | browser / NVIDIA | **109.5** | 0.03 s |
| **DiciZ** (after sprint, **dot4 default**) | hand-written WGSL, int8-activation dot products | browser / NVIDIA | **94.2** | 0.24 s |
| **DiciZ** (`?dot4=0`, exact-f32 path) | hand-written WGSL, f32 compute | browser / NVIDIA | **67.5** | 0.33 s |
| **DiciZ** (dot4) | same engine, same page | browser / **AMD 780M iGPU** | **52.9** | — |
| **ORT-Web** (transformers.js) | ONNX Runtime WebGPU | browser / NVIDIA | **36**² | — |

¹ Native numbers from the morning run (llama-bench, `-n 256 -p 0`, kernel-throughput style — no sampling or
tokenization, mildly flattering vs the browser rows). Not re-run in the evening. AMD energy was honestly
unmeasurable with `nvidia-smi`; energy rows dropped until both vendors can be measured the same way.
² Morning number; ORT-Web also tends to loop under greedy without a repetition penalty (disclosed degeneration).

**Bigger models, same engine (DiciZ, production chat path, dot4 default / f32 in parentheses):**

| Model | Device | decode t/s | TTFT (18-tok prompt) |
|---|---|---:|---:|
| Qwen2.5-1.5B q4km (loaded **from a HuggingFace URL**) | RTX 4070 | **68.7** (32.6) | 0.34 s |
| Qwen2.5-1.5B **Q2_K** | RTX 4070 | **73.7** (35.5) | — |
| Qwen2.5-7B q4km (4.68 GB on an 8 GB GPU) | RTX 4070 | **31.2** (10.2) | 0.95 s |
| Qwen2.5-7B q4km | **AMD 780M iGPU** (shared RAM) | **12.7** (4.0) | 2.56 s |
| Llama-3.2-1B / Gemma-2-2B / Phi-3.5-mini q4km | RTX 4070 (real chat UI) | **82.4 / 49.9 / 41.8** | — |

**What dot4 is (honest):** the decode matvec quantizes each activation vector to int8 per 32-block on
GPU, then uses `dot4I8Packed` integer dot products against the quantized weights (4 MACs/instruction)
— the same MMVQ/Q8_1 construction llama.cpp ships as its DEFAULT on CUDA/Vulkan. It is a precision
trade: vs the exact-f32 path, last-token logits differ by rel ~2-6e-2, top-1 stays identical in all
our tests, and texts are equivalent (benign rephrasing at most; the 7B is near-identical).
`?dot4=0` restores the exact-f32 V2 path (which itself matches the dequant reference at ~1e-6).
The batched GEMM prefill always stays full-precision f32. Validation: `tests/test_dot4.js`.

### Fairness disclosure (was missing from the morning table)
- WebLLM runs **q4f16_1** (f16 compute + activations); DiciZ computes in **f32** everywhere. Part of the
  remaining 109.5-vs-66 gap is number format, not just compiler quality. (The morning "117 vs 20 = the
  ML-compiler gap is large" conflated format + scheduling + kernels; the sprint removed the scheduling and
  kernel parts: 20 → 66 t/s.)
- The morning DiciZ "20 t/s" was also measured while a stale `llama-server.exe` held 7.5 GB of VRAM; the clean
  pre-sprint baseline was **16.3 t/s** on the auto-selected (wrong) F32 path.

### What the sprint changed (each step validated by logits-differential + identical-text tests)
1. **One `queue.submit` per token** (was ~460-540, one per GPU pass) + buffer pools (was ~950 create/destroy per token).
2. **GPU sampling**: repetition penalty + argmax on GPU — 8-byte readback per token (was ~600 KB logits + JS scan).
3. **V2 fused quantized matvec** (Q8_0/Q5_0/Q5_1/Q4_K/Q5_K/Q6_K): block header decoded once per block (was per
   element, including a `pow()` per weight), whole-u32 payload loads, native `unpack2x16float`.
4. **Batched GEMM prefill** (chunks of 8 prompt tokens, weights read once per chunk): 7B TTFT 41 s → 1.85 s.
5. Mode selection: small GGUFs now take the fused-resident path (the F32 full-GPU path was slower AND 4× bigger).

### Takeaways (updated)
- **Native Vulkan is still ~3.3× faster than WebLLM and ~5.5× faster than DiciZ** on the 0.5B — but the browser
  gap closed a lot: DiciZ went from 18× to **5.5×** of native, and from 5.8× to **1.66×** of WebLLM (which keeps
  an f16 advantage we do not use yet).
- **Cross-vendor is now REAL for DiciZ in the browser**: the same page runs on the AMD 780M iGPU (Windows
  per-app GPU preference → Chrome picks the iGPU) at 35.3 t/s (0.5B) and 4.0 t/s (7B), generating **identical
  text** to the NVIDIA runs. A 7B runs in a browser on a machine with no discrete GPU at all.
- ORT-Web LLM loses the **"load any GGUF"** wedge (needs offline ONNX conversion; rejects pre-quantized Q4 GGUF;
  4 GB browser ceiling). Reserve ORT-Web for the **video/gaming** pillars (RIFE/ESPCN — its Conv sweet spot).
- Remaining honest levers for DiciZ: shader-f16 compute, packed-dot4 integer math (`dot4U8Packed`), subgroup
  reductions — the techniques WebLLM/llama.cpp-WebGPU stack on top of what we now do.

## How to reproduce

Prerequisites: a static server for the served pages on `:8099` (e.g. `python diciz_server.py` serving the model
dir), Chrome installed, and `puppeteer-core` available (the scripts launch headless Chrome with `--enable-unsafe-webgpu`).
The browser scripts download their model on first run — point Chrome's cache to a drive with space if needed
(`run_ort.mjs` already forces the profile/cache onto `<path>` via `userDataDir`/`--disk-cache-dir`).

```bash
node run_diciz_speed2.mjs [modelUrl]  # our engine, PRODUCTION path (batched + GPU sampling + GEMM prefill); reports decode t/s + TTFT
node run_diciz_speed.mjs              # our engine, legacy harness (full-logits readback per token — kept for continuity)
node run_webllm.mjs                   # WebLLM (MLC), loads webllm_bench.html
node run_ort.mjs                      # ONNX Runtime Web (transformers.js), loads ort_llm_bench.html
```

AMD iGPU runs: create a junction (`New-Item -ItemType Junction -Path <path>\chrome-igpu -Target '<path>\Program Files\Google\Chrome\Application'`),
set `HKCU:\Software\Microsoft\DirectX\UserGpuPreferences` → `<path>\chrome-igpu\chrome.exe` = `GpuPreference=1;`,
then `DICIZ_CHROME='<path>\chrome-igpu\chrome.exe' node run_diciz_speed2.mjs`. Remove the key + junction afterwards.

Native llama.cpp (no script — direct binary), using pocketdc's bundled engine, e.g.:

```bash
# NVIDIA via Vulkan
GGML_VK_VISIBLE_DEVICES=0 ./llama-bench.exe -m qwen2.5-0.5b-q4km.gguf -ngl 99 -n 256 -p 0
# AMD iGPU via the SAME Vulkan binary
GGML_VK_VISIBLE_DEVICES=1 ./llama-bench.exe -m qwen2.5-0.5b-q4km.gguf -ngl 99 -n 256 -p 0
```

Note: pocketdc's CUDA `llama-cli.exe` crashes when stdout is **piped** (works with `> file`); the
native-NVIDIA reference here is Vulkan (CUDA ≈ similar). Energy was sampled with `nvidia-smi --query-gpu=power.draw`.

## Files
- `run_diciz_speed.mjs` / (uses the served `chat.html`) — our WebGPU/GGUF engine.
- `run_webllm.mjs` + `webllm_bench.html` — WebLLM / MLC in-browser.
- `run_ort.mjs` + `ort_llm_bench.html` — ONNX Runtime Web via transformers.js (cache forced to <path>).
