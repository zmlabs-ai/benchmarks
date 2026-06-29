# T3 gate — energy measurement, REAL (raw CSVs committed in `energy_csv/`)

**Instrument:** EnergiBridge 0.0.7 (TU Delft, MIT) over RAPL — **validation instrument only,
never a product dependency** (setup: `ENERGY_SETUP.md`, bench rig only).
**Machine:** AMD Ryzen 7 8845HS + RTX 4070 Laptop 8 GB. RAPL scope on this machine:
**CPU package only** (no DRAM domain exposed — stated, not hidden). GPU complemented by
nvidia-smi `power.draw` (measured). **Every figure below was read from the committed CSVs /
sidecars after the run** (`run_t3_energy.py`, scenario = P1-base: Qwen1.5-MoE 9.5 GB, ctx 8192).

## Measured (2026-06-11)

| phase | window | energy (CPU pkg, RAPL) | avg power | extra |
|---|---|---|---|---|
| **Boot** (plan → 9.5 GB `--no-mmap` load → first real served token) | 223.97 s | **2539.42 J** | **11.34 W** | disk-bound: the CPU mostly waits on I/O |
| **Decode** (256-token completion, same measured run wrote the token count) | 4.41 s | **215.13 J** | **48.8 W** | **0.84 J/token (CPU-side)** · GPU avg **37.4 W** (10 nvidia-smi samples) |

- Cache HIT on the serving boot (`-c 8192 -ctk q4_0 -ctv q4_0 -ngl 99 -ncmoe 10 --no-mmap`) —
  the persistence layer at work in the bench itself.
- Decode observed at ~58 tok/s in this window (256 tokens / 4.41 s, window includes the helper's
  ~0.3 s Python startup + HTTP overhead — so true decode is slightly faster than the ratio).
- RAPL is machine-wide (OS background included); windows kept short and stated.

## Counter-test (the product never depends on the instrument)
EnergiBridge made unavailable (`GOVERNOR_ENERGIBRIDGE` → nonexistent path):
`find_energibridge()` → None, `measure()` → None (**energy absent, never estimated**), and the
same 256-token completion still served with **zero errors and zero warnings**.

## What exists in the product (optional, observation-only)
`vrampilot/energy.py` — `EnergyProbe` implements the frozen `core.Probe` contract
(`Measurement(source="measured")`); activated only when EnergiBridge is detected AND functional
at startup; the CLI then records an `energy-observation` event in the store (cpu W + gpu W) and
says so. **No governor action depends on it** — the "watts budget / battery mode" degradation
axis stays out of scope until the probe has accumulated real data in the store (noted as a
future candidate, not implemented).
