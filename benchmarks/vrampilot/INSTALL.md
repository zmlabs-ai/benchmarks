# P4 gate — zero-prerequisite first run, REAL (raw log: `p4_gate_raw.txt`)

**Claim tested:** `python -m vrampilot.cli <model.gguf>` with **nothing installed** — no
llama-server anywhere on PATH, no `$LLAMA_SERVER`, an empty cache — must fetch the right pinned
binary for this OS+GPU, verify it, boot, and serve.

**Environment (the honest scope):** this machine, with the environment scrubbed by the harness
(`validation/run_p4_gate.py`): PATH reduced to Python + `<path>\Windows\System32` (+Wbem), fresh
empty `LOCALAPPDATA`, `LLAMA_SERVER` unset. It is **not** a factory-reset Windows VM — Python
and the NVIDIA driver are the only assumed pieces (both are stated prerequisites). A true
clean-VM pass remains TODO and is labeled as such below.

## What happened (2026-06-11, every line from the run)

```
target: windows-default (NVIDIA GPU -> Vulkan build (one binary, any vendor)); pinned llama.cpp b9592
downloading llama-b9592-bin-win-vulkan-x64.zip ...
  sha256 OK: 126667a2b89892fd...
llama-server: <fresh LOCALAPPDATA>\inference-autopilot\llama\b9592\llama-server.exe
[cache] nothing persisted for this (machine, model, ctx) — planned from scratch, now persisted
[plan] attempt 0: BOOTED
BOOTED in 5.29s. OpenAI endpoint: http://127.0.0.1:8080/v1   (config: -c 2048 -ngl 99)
real reply via the fetched server: 'Hello there! How can I help you today?'
P4 GATE PASSED in 7.6s (download -> sha256 -> boot -> serve, zero prerequisites)
```

- One command, **7.6 s** from cold to a real served completion (Qwen2.5-1.5B, ctx 2048,
  Vulkan on the RTX 4070 — the vendor-agnostic default build).
- The SHA256 check is **mandatory**: the unit harness also proves the tamper path — a wrong
  hash deletes the download and aborts loudly (no "best effort" fallback).
- The manifest (`autopilot/server_manifest.json`) pins llama.cpp **b9592** with hashes computed
  from real downloads of the official release; 7 targets: win vulkan/cuda(+cudart)/cpu,
  linux vulkan/cpu, macOS arm64/x64 (Metal). Official releases ship no Linux-CUDA build —
  Linux/NVIDIA gets Vulkan, stated in the manifest comment.
- `GOVERNOR_SERVER_BASE_URL` allows a private mirror (R2): the pinned hashes still apply, so a
  mirror cannot substitute binaries.

**Re-pass after security hardening (same day):** the adversarial review led to 4 fixes
(archive-member traversal guard, ARM64 targets + clear messaging, PATH-shadowing warning,
manifest-override documented as non-security). The gate was re-run on the hardened code and
passed again: BOOTED in 3.03 s, real reply served, 4.7 s total (warm OS file cache).

## Status vs the roadmap's P4 list (honest)
| item | state |
|---|---|
| manifest-pinned fetch + mandatory SHA256, OS+vendor matrix | **DONE, gate passed** (this file) |
| `.tar.gz` assets (macOS/Linux current format) handled | DONE (the old fetcher only knew `.zip` — it would have failed on today's macOS/Linux releases) |
| packaged binary bundles the manifest (`--add-data`) + selftest proves it | DONE in `release.yml`; selftest extended. CI run itself = next push |
| clean Windows VM end-to-end | ~~UNVALIDATED~~ → **VALIDATED 2026-06-11** (see the clean-VM section below) |
| R2 as primary host | founder-gated (Soleau privacy stance); mechanism ready via `GOVERNOR_SERVER_BASE_URL` |
| Azure Trusted Signing (`zmlabs-signing`) of the installer | founder-gated; not attempted |

## Clean Windows VM — VALIDATED (2026-06-11, raw log: `p4_gate_raw_cleanvm.txt`)

**Setup, 100% CLI:** VirtualBox 7.2.6 on the host; brand-new VM (EFI + emulated TPM 2.0, 4 GB
RAM, 4 vCPU); **official Windows 11 25H2 ISO** (Fido-fetched Microsoft link), `VBoxManage
unattended install` (local account, Guest Additions), provisioning via `guestcontrol`:
per-user Python 3.11 (no admin), the repo (git archive), the 1 GB test GGUF. **No GPU** in the
VM — which makes it the perfect worst case: the manifest must fall back to `windows-cpu`.

**The pass found two REAL first-run blockers that the scrubbed-host test could not see:**
1. **Fresh Windows has a lazy TLS root-CA store** → the very first `urllib` download dies with
   `CERTIFICATE_VERIFY_FAILED`. Fix shipped: fall back to the OS `curl.exe` (SChannel fetches
   missing roots automatically), stated in the output; safe by design — the pinned SHA256 is
   the security, not the transport.
2. **Fresh Windows lacks the MSVC runtime** (`vcruntime140.dll`, `vcruntime140_1.dll`,
   `msvcp140.dll`) that the official llama.cpp Windows binaries import → instant, silent
   `STATUS_DLL_NOT_FOUND` (upstream zips bundle `libomp140` but not these three). Fix shipped:
   the package bundles the three redistributable DLLs (`vrampilot/runtime_dlls/`, licenses in
   its README) and app-locally deploys them next to the fetched server only when missing.

**Final run in the clean VM (every line from the gate):**
```
GPU   : no GPU detected  (CPU, 0.0/0.0 GB free, none)
target: windows-cpu (no GPU detected -> CPU build); pinned llama.cpp b9592
downloading llama-b9592-bin-win-cpu-x64.zip ...   sha256 OK: 2b3d4e167be290bf...
runtime: copied 3 DLL(s) next to the server (vcruntime140.dll, vcruntime140_1.dll, msvcp140.dll)
[plan] attempt 0: BOOTED
BOOTED in 10.04s. OpenAI endpoint: http://127.0.0.1:8080/v1   (config: -c 2048 -ngl 1)
real reply via the fetched server: 'Hello, world!'
P4 GATE PASSED in 12.2s (download -> sha256 -> boot -> serve, zero prerequisites)
```
Honest scope: Python 3.9+ remains the one stated prerequisite of the pip distribution (the
PyInstaller binaries remove even that); the VM ran the same `run_p4_gate.py` harness as the
host pass (scrubbed PATH, fresh cache, no `LLAMA_SERVER`).

**Update (T2, 2026-06-11, append-only):** founder decision — neither R2 nor Azure. **GitHub
Releases is the primary host by design** (host-independence: the pinned manifest + mandatory
SHA256 is the security, the host needs no trust). `GOVERNOR_SERVER_BASE_URL` remains the
sovereignty mechanism (any self-hosted mirror). Windows signing postponed sine die (pointless for
a pip/pipx tool; the day a Tauri installer exists: Certum OV ~70€/yr or winget). The two
founder-gated rows above are therefore CLOSED as "won't do", not pending. T2 gate re-run logs:
`p4_gate_raw_t2.txt` (fetch from GitHub Releases under the renamed package).
