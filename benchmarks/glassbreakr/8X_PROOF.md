# GlassBreakr — « 8× » : la preuve définitive (pour ne plus jamais douter)

**Verdict : « Up to 8× FPS » est VRAI et PROUVÉ — comme débit de frames AFFICHÉES (output), exactement comme NVIDIA DLSS Frame Generation et Lossless Scaling.**
ULTRA = **1 frame captée + 7 frames interpolées par IA = 8 frames affichées** par frame captée.

---

## Les 3 preuves indépendantes

1. **Code — désassemblage du binaire livré** (`GlassBreakr_Ultra.exe`)
   `TIER_MULTIPLIER = {FREE:4, TURBO:6, ULTRA:8}` ; `compute_timesteps(N)` émet **N-1 frames distinctes** (pas de doublon) ; `multiplier = (frames_captées + frames_interpolées) / frames_captées`. → le moteur émet réellement 8 frames par frame captée en ULTRA.
   *Source : `glassbreakr-proofs/evidence/disassembly/DISASSEMBLY.md` + binaire `evidence/shipped-binary/GlassBreakr_Ultra.zip`.*

2. **Mesure fraîche sur RTX 4070** (2026-06-24, DirectML, **le vrai modèle** `rife_nano_s_flow_320x192.onnx`)
   Générer les **7 frames interpolées** d'une paire à 1080p = **15,8 ms** (médiane). → le moteur **soutient le 8× en temps réel jusqu'à ~63 fps captées** : un jeu à 30 fps → **~240 fps OUTPUT (8×)** ; à 63 fps → **~506 fps OUTPUT**.
   *Source : `geo/validation/framegen_8x_rtx4070_2026-06-24.json` (repro `measure_ultra_8x.py`).*

3. **Captures in-game (Cyberpunk 2077)**
   OUTPUT overlay **256-283 fps** en ULTRA (rendu jeu ~30-45 fps en parallèle).
   *Source : `glassbreakr-proofs/evidence/cyberpunk-run/`.*

---

## La définition honnête (la même que tout le frame-gen)

- Les frames sont **interpolées par IA** (7 sur 8). Le **rendu du jeu reste inchangé** ; la latence d'input **monte de ~15-25 ms**.
- **FPS OUTPUT (affichées) : jusqu'à 8×.** ✅
- **Frames réelles / responsives : ~2×** (Subway Surfers 30→63 sur 11 500+ frames ; Unity POOLS 59,5→68,5 réel).
- **Visible : plafonné par l'écran** (256 générées → ~180 affichées sur un 180 Hz ≈ **~4× visible**).
- C'est **exactement** le modèle de **DLSS Frame Generation** et **Lossless Scaling**. La FAQ du site le divulgue (« Output FPS = captured + AI-generated ; this is frame interpolation »).

## Règle (fin de la confusion)
« Up to 8× FPS » est **honnête et prouvé** comme débit de frames générées, **à condition** de garder la divulgation frame-gen.
**Ne JAMAIS** présenter le 8× comme de la *performance de rendu* ou de la *responsivité* — ça, c'est ~2×. Le vrai différenciateur unique = **tourner sur l'iGPU dormant** (impact dGPU <10 %).

*Mesuré et consolidé le 2026-06-24 sur RTX 4070. Sources croisées : glassbreakr-proofs (disassembly + Cyberpunk), zmlabs-glassbreakr (BENCHMARK_RESULTS.md, benchmark_pools.json), geo/validation/BENCH.md.*
