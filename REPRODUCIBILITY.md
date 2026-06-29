# Reproducibility

These benchmarks are **reproducible with the exact weights and hardware below** — not "reproducible in the abstract". Honesty first: a number is only replayable if you run the *same* quantization on the *same* class of machine.

## Headline weight — memown 30B-A3B on an 8 GB GPU (`benchmarks/memown/e1_baseline.json`)

The `~18–21 tok/s robust / ~22 best` decode result is measured on **Qwen3-30B-A3B in `Q4_K_M`**. The exact file used:

| Field | Value |
|---|---|
| Model | Qwen3-30B-A3B (MoE, 48 layers, 128 experts / 8 active) |
| Quantization | **Q4_K_M** (GGUF) |
| File | `Qwen3-30B-A3B-Q4_K_M.gguf` |
| Size | **18,632,184,480 bytes** (18.63 GB) |
| **SHA-256** | `a015794bfb1d69cb03dbb86b185fb2b9b339f757df5f8f9dd9ebdab8f6ed5d32` |
| Hardware | RTX 4070 **Laptop** 8 GB (the founder's machine) |
| Runtime | llama.cpp, expert-offload (`--n-cpu-moe` sweep), `-ngl 99 -fa 1 -ctk q8_0 -ctv q8_0 -c 4096 --no-mmap`, batch-1, decode-only |

> ⚠️ The weight is **not** included in this repo (18.6 GB — too large for a benchmarks repo). It is **archived separately** by the maintainer. The SHA-256 above is the canonical identity: a re-downloaded GGUF only reproduces the numbers if its SHA-256 matches (a different re-quant of the same model can differ slightly). To get this exact quant if lost: it is published in GGUF form on Hugging Face (search `Qwen3-30B-A3B GGUF`, file `*Q4_K_M.gguf`, e.g. the `unsloth/` or `bartowski/` GGUF repos) — verify the SHA-256 after download, and re-measure on an RTX 4070-class 8 GB GPU.

## diciz WebGPU numbers (`benchmarks/diciz/`)

The diciz in-browser figures (Qwen2.5-0.5B / 1.5B / 7B, Q4_K_M GGUF / q4f16) are reproducible in Chrome WebGPU on an RTX 4070 Laptop 8 GB — these models are small and freely re-downloadable from Hugging Face. Conditions (browser, greedy, 128 tokens, n=1) are in `benchmarks/diciz/README.md`.

## Note

Re-measure on matching hardware. "RTX 4070" in these files means the **Laptop** 8 GB part (the founder's machine), not a desktop 4070. Numbers from a different GPU or a different quant are *not* a reproduction of these results.
