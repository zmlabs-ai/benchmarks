# AMD validation — on a REAL AMD GPU (no mocks, cross-vendor)
*2026-06-07. Founder's own machine: AMD Radeon 780M iGPU (Vulkan), via prebuilt llama.cpp Vulkan build (b9550), `GGML_VK_VISIBLE_DEVICES=1`. Model: Qwen2.5-1.5B-Instruct-Q4_K_M. Reproducible via `validate_amd_local.py`.*

## Why this matters
The first validation was NVIDIA/CUDA only. AMD uses a different stack (Vulkan/ROCm) and reports OOM differently
("device memory allocation failed", not "CUDA out of memory"). This proves the autopilot — and its OOM-recovery —
work on AMD too, so the "any GPU / vendor-agnostic" claim is honest, not assumed.

## Raw results (real Radeon 780M)
**GPU seen by the Vulkan server:** `Vulkan0: AMD Radeon(TM) Graphics (8055 MiB, 7652 MiB free)` ✓

**TEST 1 — autopilot runs + serves on AMD**
```
model Qwen2.5-1.5B (0.99 GB, 28 layers) | AMD Radeon 780M (~7.6 GB)
plan: full GPU offload -> -c 8192 -ngl 99
attempt 0 -> BOOTED ; SERVED on AMD: 'OK'
```

**TEST 2 — OOM-recovery on AMD (deliberately -ngl 99 -c 1000000)**
```
attempt 0  -c 1000000                       -> OOM ; backoff: KV f16  -> q8_0 (keep context)
attempt 1  -c 1000000 -ctk q8_0 -ctv q8_0   -> OOM ; backoff: KV q8_0 -> q4_0 (keep context)
attempt 2  -c 1000000 -ctk q4_0 -ctv q4_0   -> OOM ; backoff: ctx 1000000 -> 500000
attempt 3  -c 500000  -ctk q4_0 -ctv q4_0   -> BOOTED
RECOVERED on AMD at: -ngl 99 -c 500000 -ctk q4_0 -ctv q4_0 ; served a real reply
```
=> The recovery loop caught the **Vulkan/AMD** OOM (not a CUDA error), backed off multi-axis (KV-quant first to keep
context, then shrink context), and recovered to a 500K context — on the founder's own iGPU. The OOM-detector now
covers NVIDIA (CUDA), AMD/Intel (Vulkan), and AMD (ROCm) error strings.

## Status
Validated on **NVIDIA RTX 3070 (CUDA, RunPod)** and **AMD Radeon 780M (Vulkan, local)**. Vendor-agnostic claim is now real.
Remaining: Apple Silicon (Metal) — cannot be cloud-tested; would need a Mac.
