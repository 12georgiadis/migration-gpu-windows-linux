# Migrer un poste de travail GPU NVIDIA grand public de Windows vers Ubuntu Linux

**Étude de décision, juin 2026.** Cas concret : une petite flotte de tours GPU grand public (RTX 5090 Blackwell 32 Go et RTX 4090 Ada 24 Go) utilisées pour de la génération et de l'entraînement IA, du montage et de l'étalonnage, qu'on envisage de basculer sous Ubuntu avec la posture logicielle "NVIDIA developer" des stations pro.

La question n'est pas "Linux c'est mieux ?" dans l'absolu. C'est : *qu'est-ce que je gagne, qu'est-ce que je perds, combien de temps ça prend, et est-ce que je peux y vivre définitivement ?* Réponses ci-dessous, versions vérifiées, sans angélisme.

> Ce dépôt est volontairement générique. Il ne contient aucune donnée d'infrastructure réelle (adresses, noms de machines, identifiants). C'est un retour d'expérience réutilisable.

---

## Verdict en 30 secondes

1. Le **calcul** (ComfyUI, entraînement LoRA, génération vidéo Wan/LTX, LLM locaux) tourne **mieux** sous Linux. Ce n'est pas un pari, c'est l'état de l'art : Linux est la plateforme de référence de PyTorch/CUDA.
2. La **finition** (Topaz, Adobe) **n'existe pas** nativement sous Linux. C'est l'ancre dure, et elle est en général concentrée sur **une seule** machine (le poste de travail créatif), pas sur les rigs de calcul.
3. Donc la bonne décision n'est pas "je migre toute la flotte" mais **"laquelle, dans quel ordre"** : on commence par le rig de calcul (le plus proche d'une machine Linux), on garde le poste de finition sous Windows ou en dual-boot, et on relègue la finition Adobe/Topaz vers un Mac si on en a un.
4. La "panoplie des stations à 100 k€" (Ubuntu + CUDA + containers NGC) est **gratuite** et reproductible sur du matériel grand public. Mais c'est une **optimisation**, pas une nécessité : une install Windows qui marche déjà n'est pas un problème à résoudre.
5. Recommandation : **pilote réversible en dual-boot sur le rig de calcul**, mesure 2 à 4 semaines, et laisser les chiffres décider de la généralisation.

---

## 1. Lever un malentendu fréquent : B200, "100 k€", "stations pro"

- Le **B200 est un GPU NVIDIA** (Blackwell datacenter), pas Intel. Le RTX 5090 est sa cousine grand public (même architecture Blackwell, déclinaison GeForce).
- Ce qui coûte ~500 k€, c'est une **station DGX B200 complète** (8 GPU, ~1,4 To de mémoire GPU, NVLink/NVSwitch, réseau InfiniBand), pas une carte seule.
- **Ce qui est vrai et récupérable** : les shops pro ne tournent pas sous Windows mais sous **Linux** (DGX OS est dérivé d'Ubuntu), avec un stack discipliné (driver, CUDA, cuDNN, containers NGC, environnements reproductibles). **Cette discipline logicielle est gratuite et applicable à du matériel grand public.**
- **Ce qui ne se transfère PAS** : NVLink/NVSwitch, "8 GPU vus comme un seul", mémoire HBM, FP4 datacenter, ECC, MIG, fabric InfiniBand. Plusieurs tours grand public restent **plusieurs machines indépendantes**, aucun OS n'y change rien.

La migration Linux donne la **panoplie logicielle** des pros, pas leur **topologie matérielle**. C'est déjà beaucoup.

---

## 2. La cible technique vérifiée (Ubuntu + Blackwell, juin 2026)

**Distro** : Ubuntu 24.04.x LTS **+ kernel HWE** (pas la 25.04 non-LTS). Le kernel GA d'origine (6.8) ne connaît pas le RTX 5090 ; la 24.04.4 (février 2026) embarque le kernel **6.17**.
```bash
sudo apt install linux-generic-hwe-24.04 && sudo apt dist-upgrade && uname -r
```

**Driver NVIDIA** : pour Blackwell (RTX 5090), les **modules kernel "open" sont OBLIGATOIRES**. Le paquet sans le suffixe `-open` ne voit pas la carte (`nvidia-smi` répond "No devices found"). Le RTX 4090 (Ada) accepte les deux et reste la cible Linux la plus mûre.
```bash
sudo add-apt-repository ppa:graphics-drivers/ppa && sudo apt update
sudo apt install nvidia-driver-580-open       # le suffixe -open est non négociable pour Blackwell
sudo apt-mark hold nvidia-driver-580-open     # geler la version (anti-casse au prochain kernel)
```

**CUDA + cuDNN + PyTorch** : CUDA 12.8 a été le socle Blackwell éprouvé ; en 2026 les drivers exposent déjà CUDA 13.x. **PyTorch 2.7+ avec wheels cu128 a résolu** le piège "sm_120 not compatible" de début 2025. Règle au jour J : prendre la wheel qui matche la version CUDA, et **tester chaque extension** (flash-attn, sageattention, triton) une par une.
```bash
pip install torch==2.7.0 --index-url https://download.pytorch.org/whl/cu128
```

**Posture "DGX-like" (le vrai arsenal pro, transférable)** :
```bash
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker
docker pull nvcr.io/nvidia/pytorch:25.xx-py3      # image NGC NVIDIA-tunée, reproductible
```
NGC + `nvidia-container-toolkit` + NVIDIA AI Workbench = environnements GPU versionnés et jetables. C'est la pièce qui isole CUDA/cuDNN de la dérive de l'hôte, et la raison n°1 d'adopter les containers.

**Note config single-GPU** : la doc 2026 signale des crashes Linux en **double 5090 dans une seule machine** (bug firmware GSP). Un seul 5090 par machine est la configuration **sûre** ; le bug ne concerne pas les flottes à un GPU par tour.

---

## 3. Portabilité logiciel : ce qui persiste, ce qui se contourne, ce qui est perdu

**PERSISTE (souvent meilleur)** / **CONTOURNEMENT** / **PERDU**.

### Calcul / IA (le cœur de ce qui tourne sur les rigs GPU)
| Outil | Verdict | Note 2026 |
|---|---|---|
| ComfyUI (+ custom nodes, sageattention, xformers, GGUF, WanVideoWrapper, LTXVideo, SeedVR2) | **PERSISTE (meilleur)** | xformers/sageattention/flash-attn/Triton se buildent plus proprement. Linux = environnement de référence. |
| Pinokio | **PERSISTE** | `.deb` officiel, testé Ubuntu 24.04. |
| Ostris ai-toolkit (LoRA Flux/Qwen/Z-Image) | **PERSISTE (meilleur)** | Cible de dev primaire. Contrainte = VRAM, pas l'OS. |
| WanGP / Wan 2.2 / LTX-2.3 | **PERSISTE** | PyTorch/diffusers, natifs. |
| Ollama / vLLM | **PERSISTE (meilleur)** | first-class Linux ; vLLM est quasi Linux-exclusif. |
| PyTorch / diffusers / HF | **PERSISTE (référence)** | Fin de la "taxe WSL". |

### Finition / création
| Outil | Verdict | Alternative Linux | Note |
|---|---|---|---|
| **Topaz Video AI / Photo AI** | **PERDU** | SeedVR2, SUPIR, Real-ESRGAN, chaiNNer | Pas de version Linux (beta abandonnée en 2024). Seul Nyx (denoise temporel) est un manque difficilement substituable, à louer en burst sur Windows/Mac. |
| **Adobe CC** (Pr / Ae / Ps / Ai / Id) | **PERDU** (Ps = CONTOURNEMENT fragile) | DaVinci Resolve, Blackmagic Fusion, Krita, GIMP, Inkscape, Scribus, darktable | Aucune version Linux. Wine non fiable pour la vidéo. Voir le guide souveraineté pour le détail Wine/VM. |
| **DaVinci Resolve Studio** | **PERSISTE (réserves)** | natif | Installable sous Ubuntu (`makeresolvedeb`). H.264/H.265 via NVENC/NVDEC NVIDIA uniquement : tester ses rushes réels. |
| Blender 5.x | **PERSISTE (meilleur)** | natif | CUDA/OptiX excellents. |
| Nuke / Fusion | **PERSISTE / CONTOURNEMENT** | Fusion natif Ubuntu ; Nuke officiellement Rocky 9 | Fusion couvre le compositing nodal en natif. |

### Infra / remote / dev
| Outil | Verdict | Note |
|---|---|---|
| Tailscale / WireGuard | **PERSISTE** | natif. |
| Docker / containers | **PERSISTE (meilleur)** | hôte natif, GPU passthrough propre, pas de Docker Desktop. |
| Claude Code, CLI agentiques | **PERSISTE (meilleur)** | natif, **fin de WSL**. |
| AnyDesk | **PERSISTE** | client Linux. |
| **Parsec (host)** | **CONTOURNEMENT** | host Linux non supporté. Remplacer par **Sunshine + Moonlight** (NVENC), NoMachine, RustDesk. |
| ffmpeg / git / Playwright | **PERSISTE** | natifs, meilleurs. |

**Les deux seules vraies pertes : Topaz et Adobe.** Tout le reste persiste ou s'améliore.

---

## 4. Les PLUS

1. **Perf et stabilité du calcul** : compilation native, moins de couches (pas de WDDM, pas de WSL), souvent 10 à 20 % de mieux et surtout moins de plantages exotiques (les galères SSL/protobuf/verrous de fichier sont très Windows).
2. **Reproductibilité par containers** : chaque app dans son environnement CUDA versionné et jetable.
3. **Fin de la taxe WSL** : Claude Code, Docker, tooling Python deviennent natifs.
4. **Coût logiciel : 0 €** : l'arsenal des stations pro sans le capex hardware.
5. **Headless-first natif** : Linux est conçu pour les serveurs sans écran ; Windows le tolère.
6. **Réversibilité totale via dual-boot** : on ne détruit rien pour essayer.
7. **Souveraineté** : zéro télémétrie par défaut, pas de compte propriétaire obligatoire, auto-hébergement possible de bout en bout (voir guide dédié).

## 5. Les MOINS et ce qu'on perd

1. **Topaz : perdu** (sauf machine Windows/Mac dédiée). Nyx = seul manque réel, ponctuel.
2. **Adobe : perdu** en natif. C'est la perte structurante si Premiere/After Effects sont dans la chaîne de livraison.
3. **Courbe d'apprentissage** : gérer soi-même driver, kernel, modules. Moins "ça marche tout seul".
4. **Maintenance driver/kernel** : ~1 update kernel toutes les 4 à 8 semaines, ~1 sur 5-10 casse le module NVIDIA (dkms). Budget honnête : **2 à 4 h/an** de réparation, quasi zéro sous Windows. Parades : `apt-mark hold` du driver, garder une entrée GRUB sur le kernel précédent, vérifier `dkms status` + `nvidia-smi` après chaque update.
5. **Écosystème "confort"** : jeu, petits outils Windows-only, certains pilotes périphériques.
6. **Custom nodes exotiques** : certains ne sont testés que sous Windows.
7. **Parsec host à remplacer** (Sunshine+Moonlight).

---

## 6. Combien de temps

Hypothèse clé qui réduit le coût : **les gros volumes de données ne bougent pas.** Le driver NTFS de Linux (`ntfs3`, mûr en 2026) lit et écrit les disques NTFS. On reformate uniquement le disque OS ; les disques de données se montent tels quels.

- **Rig de calcul (pilote)** : install Ubuntu + driver + CUDA + container toolkit ≈ 0,5 à 1 jour ; réinstaller ComfyUI/Pinokio/Ollama + repointer les modèles ≈ 1 à 2 jours ; remettre les custom nodes à niveau (la partie pénible) ≈ 0,5 à 2 jours. **Parité ≈ 2 à 4 jours effectifs**, traîne sur les nodes rares. En dual-boot : risque nul.
- **Poste de finition (cas dur)** : migration "définitive" déconseillée (Topaz + Adobe + gros stockage média). Dual-boot au plus (1 à 2 jours), migration complète seulement après avoir relogé la finition ailleurs.

**Total pour un pilote concluant : un week-end de travail + 1 à 2 semaines de mesure.** Pas un chantier de mois, à condition de ne pas vouloir tout faire d'un coup.

---

## 7. Peut-on migrer DÉFINITIVEMENT ?

| Fonction | Définitif sous Linux ? |
|---|---|
| Génération IA, entraînement LoRA, LLM locaux | **Oui, et c'est mieux.** |
| Étalonnage (DaVinci Resolve) | **Oui**, réserves codecs. |
| Compositing (Blender, Nuke, Fusion) | **Oui.** |
| Upscale / restauration | **Oui à ~90 %** (SeedVR2/SUPIR), sauf Nyx en burst. |
| **Finition Adobe** (Pr/Ae/Ps/Ai/Id) | **Non en natif.** Blocage structurel. |

**La vérité nette** : on peut faire tourner **le calcul définitivement sous Linux**, **pas la finition Adobe** tant qu'elle est dans le workflow. Une flotte 100 % Linux sans aucune machine Windows ni Mac n'est pas possible tant qu'Adobe y reste.

**La sortie propre** : tours GPU → Linux (usine à calcul) ; finition Adobe/Topaz → un Mac (où elles sont chez elles). Dans ce schéma, "définitivement Linux" pour les rigs GPU devient vrai, parce que la finition n'a jamais eu à y vivre. Ce n'est pas une perte, c'est un rangement.

---

## 8. Recommandation

1. **Pas de big-bang.** Ne pas migrer toutes les machines en même temps.
2. **Pilote sur le rig de calcul** (pas de Topaz/Adobe = candidat idéal), en **dual-boot** d'abord (réversible).
3. **Mesurer 2 à 4 semaines** en usage réel : vitesse, stabilité, confort. Comparer honnêtement avec l'existant Windows.
4. **Garder le poste de finition sous Windows** (ou dual-boot) tant que la finition n'est pas relogée.
5. Si concluant, **généraliser au calcul** ; sinon **revenir en arrière** (c'était réversible), à coût quasi nul.

**La part honnête** : cette migration est séduisante, techniquement saine, alignée avec une logique de souveraineté, mais c'est une **optimisation, pas une nécessité**. Une install Windows qui tourne déjà n'est pas un problème. Ne pas laisser la migration devenir du yak-shaving d'infra qui mange l'attention créative. Le pilote dual-boot est l'investissement juste : assez pour décider sur preuve, assez petit pour ne rien risquer.

---

## 9. Pièges (à vérifier)

- **Driver open obligatoire (Blackwell)** : sans le suffixe `-open`, écran noir.
- **Secure Boot** : le désactiver, ou faire l'enrollment MOK, sinon le module ne charge pas.
- **Codecs DaVinci** : H.264/H.265 via NVENC/NVDEC NVIDIA uniquement, tester ses rushes réels.
- **Ne jamais reformater un disque de données** : monter les NTFS en lecture d'abord, vérifier.
- **Parité avant bascule** : ne déclarer "migré" qu'après avoir rejoué ses workflows réels, chronos à l'appui.

---

## Documents

- `README.md` : cette étude de décision.
- `GUIDE-SOUVERAINETE.md` : alternatives open-source / auto-hébergées à Adobe, Microsoft et Parsec, et la réalité d'Adobe sous Wine (vérifiée).
- `PILOTER-AGENTS-CLAUDE-CODE-LINUX.md` : piloter plusieurs agents Claude Code sur Linux (cmux est macOS-only) et les alternatives natives (Zellij, Claude Squad).

## Licence

Texte sous **CC BY-NC-ND 4.0**. Extraits de code (commandes shell) sous **PolyForm Noncommercial 1.0.0**. Voir `LICENSE`.

*Étude basée sur un inventaire matériel réel et une recherche web vérifiée de juin 2026. Les versions logicielles évoluent : revérifier au moment d'agir.*
