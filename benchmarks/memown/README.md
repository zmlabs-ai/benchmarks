# memown — benchmarks

> Run a large local AI on a small machine. Capacity first — never reported as speed. Live: https://memown.com

## What it does
memown is a local inference layer whose goal is to **fit and serve models that are larger than your VRAM** — keeping the bulk of a Mixture-of-Experts model in system RAM (and, under memory pressure, surviving by compression) instead of demanding a bigger GPU. Its headline is a **capacity** result — "run a large model on a small machine" — explicitly **not** a tokens-per-second claim.

The measured evidence below backs that claim on real hardware: a **120B-class MoE that boots, serves and decodes on the 8 GB laptop GPU itself** — it **runs at all even with only 16 GB RAM** (experts streamed from NVMe, ~1.5 tok/s, n=1: the *worst case that still works*, **not** a nominal speed). With more system RAM the decode rises — the **same 120B reaches 7.62 tok/s with 188 GB RAM** (a server run, disclosed). The set also shows: a 30B-class MoE on the same 8 GB GPU at ~18–21 tok/s; the VRAM-vs-NVMe bandwidth gap behind the memory tier; the compression-backed OOM-survival mechanism; and the honest quality cost of memory tiering.

## Measured benchmarks
| Claim | Measured value | Conditions (model · quant · hardware · batch · n · warm/cold) | Source file |
|---|---|---|---|
| **A 120B-class MoE runs on an 8 GB laptop GPU** | boots, serves **and decodes at ~1.50 tok/s** (n=1; prefill 122 tok @1.12 tok/s, decode 300 tok @664 ms/tok); coherent generation re-confirmed 2026-06-29 | gpt-oss-120b MXFP4 (~63.4 GB, 3 shards), RTX 4070 Laptop **8 GB** + 16 GB RAM, llama.cpp `--cpu-moe -ngl 99 -c 4096`, experts streamed from NVMe, batch-1 | [`local_120b_8gb_decode.json`](local_120b_8gb_decode.json) |
| A 30B MoE runs on an 8 GB laptop GPU | **~18–21 tok/s** robust (`-ncmoe 34`, 832 MiB headroom); **22.48 tok/s** best at a memory-fragile edge (`-ncmoe 30`, 315 MiB free — OOM risk, disclosed) | Qwen3-30B-A3B Q4_K_M (18.63 GB; 48 layers, 128 experts / 8 active), RTX 4070 Laptop 8 GB, batch-1 decode-only, ctx 4096, KV q8_0, `--no-mmap`, `-ncmoe` sweep, n=37–65 | [`e1_baseline.json`](e1_baseline.json) · [`E1_BASELINE.md`](E1_BASELINE.md) |
| A 120B MoE decodes once RAM is abundant | **7.62 tok/s** (runs 7.617 / 7.584 / 7.626); **×5.08** vs **1.50 tok/s** on a 16 GB-RAM laptop. The symbolic ≥10 tok/s target is **not** reached — disclosed | gpt-oss-120B MXFP4 (63.4 GB), RunPod **L40S 48 GB + 188 GB RAM**, `--cpu-moe`, batch-1 single-token, 3 stable runs | [`runpod_64gb_validation.json`](runpod_64gb_validation.json) |
| The memory tier the design relies on | VRAM **~12.5 GB/s** vs NVMe read **~3.5** / write **~1.0 GB/s**; 4 KB latency ~28 vs ~37 µs | RTX 4070 Laptop, PCIe Gen4 x8, direct I/O (no buffering), pinned DMA, warmup 3, n=30–50, 95 % CI | [`bench_results.json`](bench_results.json) |
| Capacity: survive OOM by compression | a **3.22 GB** logical working-set **finishes** under a **1024 MB** cgroup cap via zstd **3.05×**, output **lossless** (identical checksum); the naive arm is OOM-killed | Linux cgroup `memory.max=1024 MB`, zstd, WSL drvfs (filesystem-bound, disclosed) | [`capacity_proof.json`](capacity_proof.json) |
| Quality cost of memory tiering | perplexity **+20 %** (25.17 → 30.19; 1.1995×) — **not lossless** | GPT-2 124M, WikiText-2-raw test, 287 644 tokens / 561 windows | [`wikitext_perplexity.txt`](wikitext_perplexity.txt) |

## What these numbers do NOT prove
- **Capacity ≠ speed.** "Runs a large model on a small machine" is about *fitting and serving*, not throughput; decode on the oversized configs is modest by design.
- **Capacity even under limited RAM.** This laptop has only **16 GB RAM** and **8 GB VRAM**, yet a **63 GB** model runs by streaming MoE experts from NVMe — that *is* the capacity result. The **~1.5 tok/s** (n=1) is the honest cost of that constraint, not a speed claim; with abundant RAM the same model reaches **7.62 tok/s** (datacenter L40S + 188 GB RAM — a different machine).
- **The capacity test uses synthetic log data**, not a live LLM; the win scales with compressibility (~3.05× here) and is zero on incompressible data — stated in the file.
- **Quality is not lossless** — +20 % perplexity on a GPT-2 / WikiText-2 reference (a toy reference, disclosed).
- **The 22.48 tok/s edge is fragile** (315 MiB free, OOM risk); the robust figure is ~18–21 tok/s.
- These are **batch-1** decode numbers; no batching / throughput-serving claims.

## Reproduce
See [`../../REPRODUCIBILITY.md`](../../REPRODUCIBILITY.md) for the exact weight (Qwen3-30B-A3B-Q4_K_M, 18,632,184,480 bytes, with its SHA-256) and hardware. The 30B figure reproduces on an RTX 4070-class **8 GB** GPU with the flags above; the 120B figure needs a datacenter GPU + ≥ 64 GB RAM (`--cpu-moe`). Per-file conditions and caveats: [`../../AUDIT_SUMMARY.md`](../../AUDIT_SUMMARY.md).
