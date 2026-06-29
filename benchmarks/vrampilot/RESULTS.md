# Validation — end-to-end on a REAL GPU (no mocks)
*2026-06-07. RunPod NVIDIA RTX 3070 (8 GB). llama.cpp built from source (CUDA). Model: Qwen2.5-7B-Instruct-Q4_K_M (4.68 GB). Reproducible via `runpod_ap.py` / `validate_runpod.py`.*

## What was proven (engine + multi-axis recovery + web UI, all real)
1. **Profiling reads real hardware** — model metadata from the GGUF bytes; GPU + free VRAM from the actual card.
2. **Auto-plan + serve** — chose a fitting config and booted first try, served a real completion.
3. **Multi-axis OOM-RECOVERY (the differentiator)** — given an impossible config, it recovered by trying KV-quant FIRST (keep context), then shrinking context, and served.
4. **Web UI end-to-end** — page loads, launch-via-UI works, chat-via-UI returns a real reply.

## Raw results

**Smoke:** `/health -> {"status":"ok"}` ✓

**TEST 1 — Autopilot happy path**
```
PROFILE: Qwen2.5-7B (4.68 GB, qwen2, 28 layers) | NVIDIA RTX 3070 (7.7/8.0 GB free)
plan   : full GPU offload -> -c 8192 -ngl 99
attempt 0 -> BOOTED ; SERVED reply: 'OK'
```

**TEST 2 — OOM-recovery, multi-axis (deliberately impossible: -ngl 99 -c 262144)**
```
attempt 0  -c 262144                          -> OOM ; backoff: KV f16  -> q8_0 (keep context)
attempt 1  -c 262144 -ctk q8_0 -ctv q8_0      -> OOM ; backoff: KV q8_0 -> q4_0 (keep context)
attempt 2  -c 262144 -ctk q4_0 -ctv q4_0      -> OOM ; backoff: ctx 262144 -> 131072
attempt 3  -c 131072 -ctk q4_0 -ctv q4_0      -> BOOTED
RECOVERED + SERVED at: -ngl 99 -c 131072 -ctk q4_0 -ctv q4_0 | reply: 'OK'
```
=> It tries KV-quant FIRST to keep the requested context, only shrinking context when that isn't enough.
Result: recovered to **131072** context (4x more than a context-only back-off would reach) and served — instead of crashing.

**TEST 3 — Web UI smoke (start UI -> launch via UI -> chat via UI)**
```
UI -> http://127.0.0.1:8770
GET /            -> PAGE OK
POST /api/launch -> ok: True, port 8090, plan: full GPU offload
POST /api/chat   -> reply: 'OK'
```

## Honest notes
- This validates the ENGINE + the multi-axis recovery + the web UI end-to-end. Real, not mocked.
- Differentiator confirmed unserved vs LM Studio / Ollama / Jan (load-time prevention only; see `validation/MARKET.md`).
- Bugs found+fixed BY validation across runs (in git history): boot-detect via buffered logs -> HTTP /health; flaky launch -> setsid detach + verify; `-fa` became value-taking in 2026 llama.cpp -> removed; single-axis -> multi-axis recovery.
- Honest next: validate recovery on AMD/ROCm + Apple Silicon; 1-click installer that fetches a llama-server binary.

## Reproduce
`RUNPOD_API_KEY=... python runpod_ap.py`  (builds llama.cpp CUDA, downloads the GGUF, runs the 3 tests).
Locally: `python -m vrampilot.web` (UI) or `python -m vrampilot.cli model.gguf` — needs a `llama-server` on PATH.
