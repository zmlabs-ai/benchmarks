# P1 gate — persistence (the machine that learns), REAL runs

**Hardware:** NVIDIA GeForce RTX 4070 Laptop GPU, 8 GB (free VRAM *measured* via nvidia-smi).
Host: AMD Ryzen 7 8845HS (its Radeon 780M iGPU is present but llama-server selects CUDA0).
**Server:** llama-server `version: 9437 (aa46bda89)`, CUDA build.
**Model:** Qwen1.5-MoE-A2.7B-Chat Q4_K_M — **9.5 GB file on an 8 GB GPU** (forces expert-offload).
**Gate harness:** `validation/run_p1_gate.py`, fresh dedicated DB per run (`GOVERNOR_DB`).
Raw output: `p1_gate_raw_ctx8192.txt` (base) and `p1_gate_raw_ctx32768.txt` (hard). Every number
below was read from those logs after the runs — none was written in advance.

## Base scenario (ctx 8192) — 2026-06-11, all four steps passed

| step | cache | total time | result |
|---|---|---|---|
| A. run 1, empty DB | miss | **222.63 s** | plan attempt 0 BOOTED, real reply `'Hi there!'` |
| B. run 2, warm | **hit** | **223.7 s** | booted directly at the persisted config, real reply `'Hello!'` |
| C. fingerprint mutated (simulated driver/GPU change, labeled artificial) | miss | 224.91 s | full replan, BOOTED, reply `'Greetings!'` |
| D. cached entry poisoned with a flag that cannot boot | **stale** | 226.61 s | cache attempt died → entry **superseded (kept as history)** → replan BOOTED, reply `'Hello!'` |

Working config in every booted step: `-c 8192 -ctk q4_0 -ctv q4_0 -ngl 99 -ncmoe 11 --no-mmap`.

Final store state (append-only, from the run):
```
#4 [active]     Qwen1.5-MoE-A2.7B-Chat ctx=8192 hits=0  (re-learned after D)
#3 [superseded] poisoned-entry         ctx=8192 hits=0  flags: --definitely-not-a-flag
#2 [active]     Qwen1.5-MoE-A2.7B-Chat ctx=8192 hits=0  (mutated-fingerprint key)
#1 [active]     Qwen1.5-MoE-A2.7B-Chat ctx=8192 hits=1  (the original entry, hit by run B)
```

### Honest reading of the base numbers
- Cold 222.63 s vs warm 223.7 s: **when the planner boots first-try, the cache saves no
  wall-clock** — boot time is dominated by reading 9.5 GB from disk (`--no-mmap`), which both
  runs pay. The cache's value is **skipping failed attempts** (each costs a full model load,
  ~220 s here) and starting at the config that survived — see the hard scenario below.
- Step C demonstrates the invalidation *mechanism* with an artificially mutated fingerprint;
  a real driver update or GPU swap produces the same key change.
- Step D proves append-only supersede: the poisoned entry stays in history, is never served
  again, and the replanned config is appended as a new row.

### Bug found by this gate (and fixed before it passed)
The first gate attempt failed for a reason worth recording: `recover.py` had a fixed
`boot_timeout=120` s; loading 9.5 GB with `--no-mmap` takes longer than that, so the loop was
killing **healthy boots mid-load** and degrading configs that would have worked. Fixed: the
timeout now scales with model size (30 s/GB, floor 120 s). This is exactly the class of bug
only a real run finds.

## Hard scenario (ctx 32768) — honest result: the planner kept landing first-try

| step | cache | total time | result |
|---|---|---|---|
| A. run 1, empty DB | miss | **223.72 s** | attempt 0 BOOTED at `-c 32768 -ctk q4_0 -ctv q4_0 -ngl 99 -ncmoe 14 --no-mmap`, real reply |
| B. run 2, warm | **hit** | **221.98 s** | booted directly at the persisted config, real reply |

We expected a multi-attempt recovery trail at 32k context; the planner adapted instead
(more expert layers offloaded: `-ncmoe 14` vs `11` at 8k) and booted first-try. Two honest
conclusions, not one:
1. **On this machine/model the planner is conservative enough to land first-try** even at the
   model's full training context — itself a useful result (32k ctx, 9.5 GB MoE, 8 GB GPU).
2. Therefore the cache's measured win here is consistency, not wall-clock. **The wall-clock win
   exists exactly when an attempt fails**: a failed attempt costs a full model load
   (~220 s measured per attempt on this disk), and the cache starts at the config that survived.
   Step D of the base scenario shows the mechanics (failed cache attempt → supersede → replan).
   The P2 watchdog (see `WATCHDOG.md`) persists the config that survived a runtime restart, which
   the next launch then boots directly — that is the learning loop closed.

## Known limits (stated, not hidden)
- The roadmap sketched this gate on the 780M/Vulkan machine; it ran on the same laptop's
  RTX 4070 (CUDA, *measured* VRAM). On Vulkan-only hardware the free-VRAM input to the plan is
  an *estimate* (85% of the biggest heap) — the recovery loop is the net, as documented.
- `elapsed` includes plan + every attempt + health/first-token check; it is dominated by the
  9.5 GB `--no-mmap` disk read in all runs shown.
- Steps C and D force the mechanism artificially (mutated fingerprint, poisoned entry) and are
  labeled as such; the code paths they exercise are the real ones.
