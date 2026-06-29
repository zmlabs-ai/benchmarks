# P2 gate — in-inference watchdog, REAL runs (NVIDIA, measured VRAM)

**Hardware:** NVIDIA GeForce RTX 4070 Laptop GPU 8 GB (nvidia-smi, *measured*).
**Server:** llama-server b9437 CUDA. **Model:** Qwen1.5-MoE-A2.7B-Chat Q4_K_M (9.5 GB on 8 GB).
**Harness:** `validation/run_p2_gate.py`. Raw, timestamped output: `p2_gate_raw.txt`.
The roadmap sketched this gate on RunPod NVIDIA; it ran on the local RTX 4070 — same vendor,
same real nvidia-smi measurements, zero cloud spend. Every number below is read from the run.

## Forced scenario — real external VRAM pressure during a real streamed generation

The real-world event simulated: *the user opens another GPU app mid-generation*. The pressure
is a genuine second llama-server (1 GB model, full GPU offload, ctx 32768 f16 KV) loading onto
the same GPU while the governed server streams a long essay (912 streamed chunks).

Timeline (timestamps from the run):

| t | event |
|---|---|
| +199.9 s | governed server BOOTED (`-c 8192 -ctk q4_0 -ctv q4_0 -ngl 99 -ncmoe 11 --no-mmap`), **1037 MiB free** |
| +202.0 s | watchdog armed; **VRAM floor auto-calibrated: 400 MiB** |
| +208.0 s | long generation streaming; pressure server starts loading |
| +233.1 s | **SOFT ALERT** — VRAM trend −30 MiB/s toward the floor |
| +235.2 s | **CRITICAL** — floor crossed: **102 MiB free < 400 MiB** → controlled restart |
| +459.0 s | **RECOVERED in 223.9 s** at degraded config (**ctx 8192 → 4096**), same port 8080 — *while the pressure server still held its VRAM* |
| +465.6 s | **real completion served after recovery**: `'Hello!'` |

- The watchdog fired **before** any CUDA crash or WDDM spill-collapse; the server never died —
  it was preemptively degraded and restarted by the governor.
- The restart reloads the model (~224 s here, disk-bound). The recovery happened **under
  sustained pressure** — the degraded config fits beside the intruder, which is the point.
- **Honest note on the "lost generation":** in this run the 912-chunk generation happened to
  finish (`ended_cleanly=True`) seconds before the restart took effect, so nothing was cut.
  In general a stream in flight at restart time IS lost; the watchdog says so in its own
  critical message. We do not claim invisible recovery.

## What was persisted (the learning loop, read back from the gate DB)

```
events:  watchdog-restart  reason='VRAM floor crossed: 102 MiB free < 400 MiB'
         vram_mib_at_trigger=102  recovery_s=223.9
         recovered_flags = -c 4096 -ctk q4_0 -ctv q4_0 -ngl 99 -ncmoe 11 --no-mmap
configs: #2 [active] survivor config (ctx 4096), server='after-watchdog-restart'  <- next boot starts here
         #1 [active] original config (ctx 8192)
```
The next launch of this (machine, model, ctx-request) boots the **survivor** config directly
(append-only: the original entry stays in history). The report shows the working flags, so the
reduced context is visible, not silent.

## Counter-scenario — zero false positives

After recovery (pressure removed): 3 normal generations on the same server — replies
`'Hello!'`, `'Hello!'`, `'Greetings!'` — **0 soft alerts, 0 interventions**.
(The 0.1 s reply times are prompt-cache hits on a repeated prompt; the replies are real.)

## Known limits (stated, not hidden)
- VRAM triggers are armed **only where free VRAM is measured** (NVIDIA). On Vulkan/Apple the
  estimate isn't good enough to act on: the watchdog honestly downgrades itself to
  process+`/health` watch and says so in its first event.
- llama.cpp cannot shrink a live server's KV cache (no hot-resize upstream as of b9437/b9592);
  the only runtime action is this controlled restart. If upstream lands hot KV resize
  (e.g. the direction of ggml-org/llama.cpp#20757), adopt it and retire the restart for that path.
- The trend metric is VRAM-vs-time, not VRAM-vs-tokens (token counts need `--metrics`, which we
  probe but do not require).
- One soft alert fired with a negative ETA (already under the floor at first computation) —
  cosmetic; fixed after the run (alerts now only fire above the floor). The critical path was
  unaffected.
