# BENCHMARKS — Manifeste (dossier `benchmarks/` du futur dépôt public ZMLabs)

Préparé pour publication. Chaque chiffre pointe vers son fichier de preuve. Tous les chemins ci-dessous sont
relatifs au dossier `benchmarks/`. Chaque fichier a été **relu** pour confirmer
des **conditions de mesure complètes** (modèle + quant + matériel + batch + n +
chaud/froid) et **rédigé** (placeholders `<path>` / `<org>` ; aucun secret trouvé).

Règle d'inclusion : seules les **preuves MESURÉES à conditions complètes** entrent.
Aucun chiffre modélisé, synthétique-sans-LLM, ou « optimiste/RAM-serveur ».

## Fichiers inclus

| Fichier | Site | Claim (mesuré) | Conditions clés | Nature |
|---|---|---|---|---|
| `vrampilot/RESULTS.md` | vrampilot | OOM-recovery multi-axe 262144→131072 puis sert une vraie réponse | RTX 3070 8 GB (RunPod), Qwen2.5-7B Q4_K_M 4.68 GB, batch-1, e2e llama.cpp CUDA, 3 tests | MESURÉ |
| `vrampilot/ENERGY.md` | vrampilot | 0,84 J/token (décode, côté CPU) ; boot 2539 J | Ryzen 7 8845HS + RTX 4070 Laptop 8 GB ; EnergiBridge/RAPL = **CPU-package only, divulgué** ; Qwen1.5-MoE 9,5 GB ctx 8192 ; décode 256 tok ; GPU 37,4 W (10 samples nvidia-smi) | MESURÉ |
| `vrampilot/WATCHDOG.md` | vrampilot | Watchdog VRAM en cours d'inférence : restart contrôlé + recovery 223,9 s sous pression réelle | RTX 4070 Laptop 8 GB (nvidia-smi mesuré), llama-server b9437 CUDA, Qwen1.5-MoE-A2.7B Q4_K_M 9,5 GB sur 8 GB | MESURÉ |
| `vrampilot/PERSISTENCE.md` | vrampilot | Cache/persistance « machine qui apprend » : cold 222,63 s vs warm 223,7 s ; le gain = sauter les tentatives ratées | RTX 4070 Laptop 8 GB CUDA, llama-server 9437, Qwen1.5-MoE 9,5 GB/8 GB, ctx 8192 et 32768 | MESURÉ |
| `vrampilot/INSTALL.md` | vrampilot | Zéro-prérequis : download→sha256→boot→serve en 7,6 s ; clean Windows VM validé | Machine de bench (PATH scrubbed) + VM VirtualBox Win11 25H2 **sans GPU (fallback CPU)** ; llama.cpp b9592 ; Qwen2.5-1.5B ctx 2048 | MESURÉ |
| `vrampilot/CROSS_PLATFORM.md` | vrampilot | Run+serve sur NVIDIA/AMD/Apple ; OOM-recovery clean sur NVIDIA+AMD (**Apple = mécanisme en place, démo sane-OOM « not yet » divulguée**) | RTX 3070 RunPod / Radeon 780M local / M1 CI macos-14 | MESURÉ |
| `vrampilot/AMD.md` | vrampilot | OOM-recovery sur AMD réel 1000000→500000 puis sert | Radeon 780M iGPU Vulkan **local, single-machine divulgué** ; llama.cpp Vulkan b9550 ; Qwen2.5-1.5B Q4_K_M | MESURÉ |
| `glassbreakr/framegen_8x_rtx4070_2026-06-24.json` | glassbreakr | 8× = **débit de frames générées** (1 captée + 7 interpolées) ; 15,8 ms médiane pour 7 frames 1080p ; soutient 8× jusqu'à ~63 fps captées | RTX 4070 Laptop ; onnxruntime 1.24.4 DirectML ; modèle RIFE-Nano flow 320×192 ONNX ; warmup 6 / médiane 30. **Scope honnête : frames interpolées, latence +15-25 ms, PAS de qualité à 8×** | MESURÉ (débit) |
| `glassbreakr/8X_PROOF.md` | glassbreakr | « 8× » = débit AFFICHÉ (output) comme DLSS FG ; responsivité ~2× ; visible plafonné par l'écran | Mesure RTX 4070 (2026-06-24) + désassemblage binaire + captures Cyberpunk | MESURÉ (débit, PAS qualité) |
| `glassbreakr/BENCH.md` | glassbreakr | Multiplicateurs FREE 4× / TURBO 6× / ULTRA 8× **générés** ; runs Subway Surfers ~2× output sur 11 500+ frames ; iGPU 270p 74,9 fps | HP OMEN Ryzen 7 8845HS + RTX 4070 Laptop + Radeon 780M ; DirectML. **Caveats internes : 8× = frame count pas grade qualité (pas de PSNR/SSIM) ; « ≈ un GPU desktop haut de gamme » = estimé, NON publié** | MESURÉ (débit) |
| `diciz/rife_native_vs_browser_2026-06-28.json` | diciz | Fait citable = RIFE-Nano **natif ~4 ms/frame** (DirectML, temps réel). Le ratio 41,26× browser/native est présent **avec `decision` = « ne pas publier le ratio »** | RTX 4070 Laptop 8 GB ; RIFE-Nano 270p fp32 ; batch-1 ; input byte-identique ; warmup 10 / N 50 médiane ; parité sortie (8 % peak div) divulguée | MESURÉ (RIGOUREUX) |
| `diciz/README.md` | diciz | Décode LLM on-device : DiciZ 94,2 t/s (NVIDIA, dot4) / 67,5 (f32) ; Qwen2.5-7B 31,2 t/s ; cross-vendor AMD 780M texte identique | Win 11 ; RTX 4070 Laptop 8 GB + Radeon 780M ; Qwen2.5-0.5B ~4-bit greedy 128 tok ; Chrome WebGPU. **Divulgation n=1 par config ; native llama-bench « mildly flattering » divulgué** | MESURÉ |
| `memown/local_120b_8gb_decode.json` | memown | gpt-oss-120B **décode 1,50 t/s sur le laptop 8 GB** (n=1) ; charge + sert + génère du texte cohérent | RTX 4070 Laptop **8 GB** + 16 GB RAM ; MXFP4 ~63,4 GB ; `--cpu-moe -ngl 99 -c 4096` ; experts streamés du NVMe ; batch-1 ; prefill 122 tok @1,12 t/s / décode 300 tok ; **capacité≠vitesse** | MESURÉ (n=1) |
| `memown/runpod_64gb_validation.json` | memown | gpt-oss-120B MoE décode **7,62 t/s** (×5,08 vs 1,50 sur laptop 16 GB) ; **seuil ≥10 NON atteint, divulgué** | RunPod L40S 48 GB VRAM + 188 GB RAM ; `--cpu-moe` ; MXFP4 120B 63,4 GB ; batch-1 mono-token ; 3 runs stables (7,617 / 7,584 / 7,626) | MESURÉ |
| `memown/e1_baseline.json` | memown | Qwen3-30B-A3B Q4 (18,63 GB) sur GPU 8 GB ; décode jusqu'à 22,48 t/s (best ncmoe 30) | RTX 4070 Laptop 8 GB (nvidia-smi) ; llama.cpp expert-offload ; ctx 4096 KV q8_0 `--no-mmap` ; decode-only ; sweep ncmoe, n_samples 37-65 | MESURÉ |
| `memown/E1_BASELINE.md` | memown | **Correction d'honnêteté du même run** : ~18–21 t/s robuste (ncmoe 34, 832 MiB de marge) / **22 t/s = bord fragile** (315 MiB, risque OOM) | RTX 4070 Laptop 8 GB ; 3 reps sur les configs clés ; survival-gated tune (marge ≥300 MiB mesurée après une vraie génération) | MESURÉ |
| `memown/bench_results.json` | memown | VRAM ~12,5 GB/s vs NVMe read ~3,5 / write ~1,0 GB/s ; latence 4 KB VRAM 28 µs vs NVMe 37 µs | RTX 4070 Laptop 8 GB, PCIe Gen4 x8 ; I/O direct (NO_BUFFERING) ; cudaMemcpy pinned ; warmup 3 ; N 30-50 ; IC95 t-distribution | MESURÉ (RIGOUREUX) |
| `memown/wikitext_perplexity.txt` | memown | RAMEXTEND (SVD+INT4) PPL 30,19 vs GPT-2 base 25,17 = +20 % (1,1995×) ; honnête **« pas lossless »** | GPT-2 124M ; WikiText-2 split test ; max_length 1024 stride 512 ; 287 644 tokens / 561 windows | MESURÉ |
| `memown/capacity_proof.json` | memown | Survie-OOM par compression : working-set logique 3,22 GB FINIT sous cap cgroup 1 GB (l'arm naïf = OOM-killed) via zstd 3,05× ; sortie lossless (checksum identique) ; **caveat capacité≠vitesse** | cgroup memory.max 1024 MB ; zscan cap 512 MB ; WSL drvfs fs-bound divulgué | MESURÉ |

## EXCLUS (fichier — raison)

- **memown `cmr_ttft_120b.json` (×52,9)** — n=1 / prompt unique / déclaré « PAS vérifié » par le fondateur → conditions incomplètes.
- **memown `runpod_8gb_reverify.json` (22,16 t/s)** — auto-labellisé « optimiste / RAM serveur 251 Go » → trompeur pour un laptop 8 Go.
- **memown `benchmark_final.txt` (7,8×)** — synthétique, sans LLM réel → « speed » trompeur.
- **memown `rtx4070_benchmark.txt` (614×)** — synthétique, sans LLM réel → « speed » trompeur.
- **Tout fichier marketing** (`gb-src/marketing/*`, GUMROAD, REDDIT, etc.) et tout claim « = GPU desktop haut de gamme / 119 FPS / 100× less money / 0,16 ms » — marketing ou estimé, jamais mesuré à conditions complètes. (Note : la comparaison d'équivalence GPU desktop a été **retirée** de `glassbreakr/BENCH.md` dans cette copie publique.)

## Note de rédaction (placeholders)

Chemins absolus et identifiants locaux remplacés par `<path>` / `<org>` / `user`.
Aucun motif de secret (clés RunPod, API LLM, GitHub PAT, Resend, Google, AWS, blocs
PEM) n'a été trouvé dans les sources. Domaines publics et URLs R2 publiques (pub-*.r2.dev)
sont conservés à dessein.

Faux positifs documentés du scan final (contenu de preuve d'origine, NON modifié) :
le motif Resend (« re » + underscore + alphanumérique) déclenche sur deux sous-chaînes
anodines — le nom du script de repro glassbreakr (suffixe « ultra_8x.py ») et la clé
JSON memown du coût horaire (suffixe « _usd »). Aucune n'est une clé Resend ; les deux
sont des éléments de preuve d'origine laissés intacts à dessein.
