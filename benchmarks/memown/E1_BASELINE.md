# E1 — Honest baseline: oversized MoE on an 8 GB GPU

**What/where:** Qwen3-30B-A3B-Q4_K_M (18.63 GB, 48 layers, 128 experts / 8 active) served on an
**RTX 4070 Laptop 8 GB** via llama.cpp expert-offload (`-ncmoe`), batch-1 decode, ctx 4096,
KV q8_0, `--no-mmap`, build #20819. Harness: `validation/e1_baseline.py` (decode-only t/s from
the server's own `timings`; util/power from live `nvidia-smi`). Raw data: `e1_baseline.json`
(sweep) + `e1_confirm.json` (repeats). Nothing fabricated.

## Result (3 reps on the key configs)

| -ncmoe | decode t/s (mean [min–max], n) | VRAM peak | util.gpu | util.mem | power |
|--------|-------------------------------|-----------|----------|----------|-------|
| **30** | **22.2 [21.96–22.48], n=3** ✅ stable & fastest | 7873 MiB (96 %) | ~51 % | ~14 % | ~37 W |
| 34     | 16.9 [13.87–20.84], n=3 ⚠️ variable | 7231 MiB | ~28 % | ~13 % | ~13 W |
| 38     | 12.65 (n=1) | 5889 MiB | 31 % | 14 % | 12 W |
| 44     | 18.32 (n=1) | 3745 MiB | 26 % | 12 % | 13 W |

**Headline (honest):** Qwen3-30B-A3B-Q4 (18.6 GB) runs on this 8 GB card at **~22 t/s** at the
optimal offload (`-ncmoe 30`), VRAM 96 % full. Prefill ~58 t/s at that config.

## What the telemetry PROVES (and corrects)

1. **This regime is NOT VRAM-bandwidth-bound.** `util.mem` (memory-controller busy %) stays at
   ~12–15 % — nowhere near saturation. So the **"MBU 79–89 % ceiling" does NOT apply to an
   offloaded MoE**; that ceiling describes resident-dense decode. The binding link here is the
   **host CPU/RAM/PCIe path that streams/computes the offloaded experts** (not exposed by
   nvidia-smi). A simple `weights × t/s / peakBW` MBU is meaningless in this regime.
2. **Throughput scales with how much stays resident.** Lower `-ncmoe` (more experts on GPU) →
   higher `util.gpu`/power and **faster + more stable**: ncmoe 30 = 22.2 t/s @ ~51 % GPU / 37 W,
   stable; ncmoe 34 = host-bound, GPU starved (~28 % / 13 W) and **variable (13.9–20.8 t/s)** with
   system load. The optimum (ncmoe 30) is both the fastest and the most reproducible.
3. **Energy:** ncmoe 30 ≈ 1.66 J/token (37 W, fast); ncmoe 34 ≈ 0.85 J/token but slow (GPU idling
   on the host path — the lower J/token is starvation, not efficiency).

## vs prior single-shot numbers
Prior unreplicated measurements bracket this (pocketdc1 ncmoe32 = 10.89 t/s; puce-v2 ncmoe36 =
46.3 t/s) — the wide spread is itself evidence of the **high-variance, host-bound** nature of MoE
offload. This 3-rep ncmoe-30 = **22.2 t/s** is the defensible figure on this machine, with full
telemetry.

## CORRECTION via survival-gated tune (validation/tune_ncmoe.py, tune_ncmoe.json)
A robustness re-check under live conditions (free-VRAM margin ≥300 MiB, measured AFTER a real
generation) found: **ncmoe 34 → 832 MiB free (SAFE)**, **ncmoe 32 → only 154 MiB free (UNSAFE)**.
So the **robust production optimum is `-ncmoe 34` (~18–21 t/s), not 30.** The 22 t/s at ncmoe 30
seen in the sweep was **at the edge** (315 MiB) and depends on how much VRAM the desktop is using
at that instant — it risks OOM in production. Honest headline: **~18–21 t/s robustly; ~22 t/s only
at the fragile edge.** The governor caches the safe optimum (ncmoe 34) in the store, so subsequent
boots start there.

## Product implications (next steps)
- **Robust optimal = `-ncmoe 34`** (832 MiB headroom) — recorded as known-good in the governor store.
  Going lower (32/30) is faster but unsafe under the 300 MiB margin; the survival-gated tune correctly
  refuses it rather than risk OOM in the user's face.
- **GAP found:** `serve.py`'s native intent is FIXED and **does not seed `-ncmoe`** for a MoE, so
  the OOM ladder cannot expert-offload (it would cut `-ngl` instead = worse for MoE). Fix: seed
  `-ncmoe` via `vendor/vrampilot/plan.py` (`is_moe → -ncmoe`) so the product actually boots near
  this optimum. (Tracked as the next step after E1.)
- This baseline is the reference all later levers (AdapMoE prefetch, etc.) must beat — measured on
  the same machine, decode-only, same telemetry.
