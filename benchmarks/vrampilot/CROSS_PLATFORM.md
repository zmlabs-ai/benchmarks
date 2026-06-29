# Cross-platform validation — honest summary (3 GPU vendors, real hardware, no mocks)
*2026-06-07.*

| Vendor | Stack | Where | Run + serve | OOM-recovery (sane config + real reply) |
|---|---|---|---|---|
| **NVIDIA RTX 3070** | CUDA | RunPod | ✅ `'OK'` | ✅ recovered 262144→131072, served real reply (`RESULTS.md`) |
| **AMD Radeon 780M** | Vulkan | founder's PC (local) | ✅ `'OK'` | ✅ recovered 1000000→500000, served real reply (`AMD.md`) |
| **Apple M1** | Metal | GitHub Actions `macos-14` | ✅ `'OK'` | ⚠️ mechanism works (boots), but a clean "sane-OOM" demo isn't feasible on the tiny 7 GB CI runner — small models fit, and forcing OOM there needs an absurd context (1M = 30× the model's 32k training) which boots but outputs garbage. A real request never hits this because `plan` caps context at the model's training limit. |

## The big claim, proven
**The autopilot runs and serves on NVIDIA, AMD, and Apple** — vendor-agnostic, on real hardware of all three (including the founder's own AMD iGPU). The OOM-detector covers CUDA + Vulkan + ROCm + Metal error strings.

## OOM-recovery, honestly
Cleanly demonstrated (recover to a sensible config + serve a real reply) on **NVIDIA and AMD**. On the Apple CI runner the recovery mechanism works, but the runner is too small (7 GB, tiny models) to stage a *sane* out-of-memory case — forcing one requires an absurd context that the model can't sensibly run anyway.

## What Apple validation gave us (real improvements, all platforms)
Two genuine bugs/limits surfaced only by Apple (the strictest backend), now fixed everywhere:
1. **Quantized KV needs flash-attention** — recovery now enables `-fa on` with KV-quant (Metal is strict; CUDA/Vulkan were lenient and hid it).
2. **`/health`=200 is not "it works"** — recovery now verifies it can ACTUALLY generate a token before declaring success, not just that the server responds.

## Honest scope
Run+serve is proven on all three vendors. Recovery is proven clean on two (NVIDIA, AMD). Apple's recovery mechanism is in place + the cross-vendor error detection works; a sane-OOM Apple demo would need a real Mac with more memory (not a 7 GB CI runner).
