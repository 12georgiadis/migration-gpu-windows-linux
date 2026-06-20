# Poste de travail autonome sur Linux : quitter Adobe, Microsoft et Parsec

**Guide, juin 2026.** Compagnon de l'étude de migration GPU Windows vers Ubuntu. Objectif : un poste créatif sans logiciel propriétaire à télémétrie, ou au minimum sans dépendance structurelle à Microsoft/Adobe/Google. Cible : créateur solo pragmatique, pas paranoïaque, qui veut sortir des écosystèmes fermés sans devenir administrateur système à plein temps.

Versions et faits vérifiés par recherche web de juin 2026 (la section Adobe a fait l'objet d'une vérification adversariale). Les versions évoluent vite : revérifier au moment d'agir.

> Note de cadrage : "autonome" ne veut pas dire "tout auto-hébergé". Le bon curseur, c'est posséder ses données et ne pas dépendre d'un acteur unique, pas réimplémenter une infra d'entreprise. Plusieurs fois dans ce guide, la recommandation pragmatique sera de NE PAS auto-héberger (mail) ou de NE PAS pousser le curseur (Qubes/Tails).

---

## 1. Quitter Adobe

C'est l'ancre la plus dure de tout le poste. Trois voies : le natif Linux, Wine, et la VM à passthrough GPU.

### 1.1 Les alternatives natives Linux, app par app

| App Adobe | Meilleure alternative native | Version (juin 2026) | Verdict |
|---|---|---|---|
| Premiere Pro | **DaVinci Resolve 21** (+ GPU NVIDIA) | 21 (3 juin 2026) | **Remplace vraiment** |
| Premiere Pro (léger) | Kdenlive | 25.12.2 | Remplace partiellement |
| After Effects (compositing) | Blackmagic Fusion (dans Resolve) | Fusion 20 | Remplace vraiment (compositing) |
| After Effects (motion design / templates) | aucun | — | **Pas d'équivalent** |
| After Effects (3D / procédural) | Blender | 5.x | Remplace partiellement |
| Photoshop | GIMP 3.2 + Krita 5.3.2 | 3.2 / 5.3.2 | Remplace partiellement |
| Lightroom | darktable | 5.4 (5.6 le 21 juin) | **Remplace vraiment** |
| Illustrator | Inkscape | 1.4.4 | Remplace partiellement |
| InDesign | Scribus | 1.6.6 | Remplace partiellement (limites sévères) |
| Audition | Reaper 7.74 / Ardour | natifs Linux | **Remplace vraiment** |

Points saillants :
- **DaVinci Resolve 21** (sorti hors beta le 3 juin 2026) est le pivot. Il remplace vraiment Premiere, ET fait l'étalonnage (le meilleur du marché), le compositing par nœuds (Fusion), l'audio (Fairlight), et même le RAW photo (nouvelle page Photo). Gratuit jusqu'en 4K UHD 60 ; Studio à 295 $ en licence à vie (paiement unique, pas d'abonnement). **Piège Linux clé : avec un GPU AMD, le decode/encode hardware H.264/H.265 est cassé ; avec NVIDIA, ça fonctionne.** Donc sur Linux + Resolve, prendre NVIDIA.
- **Le seul vrai trou : After Effects pour le motion design / templates.** Fusion et Blender couvrent ~80 % des besoins VFX/3D, mais pas l'animation de texte et les templates au quotidien. C'est l'app Adobe la plus difficile à remplacer.
- **darktable** remplace vraiment Lightroom (workflow différent, courbe d'apprentissage de 10 à 20 h). **Reaper** (60 $, natif Linux) ou **Ardour** (100 % libre) remplacent vraiment Audition.
- **GIMP 3.2 + Krita 5.3.2** couvrent le photomontage et la peinture, mais pas le Generative Fill ni le CMYK pro. **Inkscape** et **Scribus** font le quotidien vectoriel/PAO, avec leurs limites (CMYK pro partiel, Scribus ne lit pas les .indd).

### 1.2 Adobe sous Wine : la vérité 2026 (la question la plus posée)

**Est-ce que les gens le font vraiment ? Oui pour Photoshop, non pour la vidéo.**

L'événement de l'année côté Linux créatif : en **janvier 2026**, un développeur (pseudo PhialsBasement) a patché Wine pour corriger les incompatibilités MSXML3/MSHTML qui empêchaient l'installeur Creative Cloud de tourner. Résultat : **Photoshop 2021 et 2025 s'installent et fonctionnent réellement sous Wine** (couvert par Phoronix, Tom's Hardware, gHacks, OMG!Ubuntu). PS 2021 est décrit comme "robuste".

Échelle WineHQ (du pire au meilleur) : Garbage < Bronze < Silver < Gold < Platinum.

| App Adobe | Wine 2026 | Verdict honnête |
|---|---|---|
| Photoshop CC 2021 | Silver, "robuste" avec patches | **Marche** (production OK) |
| Photoshop CC 2024/2025 | Bronze/Silver (build Wine patché) | **Utilisable** (GPU via vkd3d), Camera Raw IA aléatoire |
| Illustrator CC | Bronze | Galère (gradients, effets, polices) |
| Lightroom Classic | Bronze/Silver | À moitié : le module Develop crashe (erreurs GPU) |
| Premiere Pro 2025 | **Garbage** | **Mort** : UI réécrite en Direct2D, impossible à afficher sous Wine |
| After Effects | **Garbage** | **Mort** : perfs catastrophiques, renders impossibles |
| InDesign | Bronze/Garbage | Non viable |

Détails pratiques :
- **L'outil qui marche pour les CC récentes = un build Wine patché** (scripts "wine-adobe-installers", winetricks avec vkd3d/msxml3/corefonts), **pas Wine vanilla ni CrossOver**. CrossOver ne couvre plus correctement les CC récentes (il reste bon pour CS6 et antérieurs).
- **Photoshop sous Wine : utiliser X11, pas Wayland** (Wayland casse le drag-and-drop et fait flickerer). Camera Raw et les features IA peuvent planter selon les drivers GPU.
- **Premiere et After Effects : ne même pas essayer.** Pas de GPU/CUDA, codecs et sync cassés, et PP 2025 a une UI Direct2D inaffichable sous Wine. After Effects est "encore pire". Personne ne fait de vidéo pro sous Wine.
- **Nuance autonomie importante** : Adobe sous Wine reste du logiciel propriétaire avec sa télémétrie. Wine ne te rend pas autonome, il te rend juste indépendant de Windows. Ce n'est pas la même chose.

### 1.3 L'option "garder Adobe mais autonome" : VM Windows + GPU passthrough

C'est la seule voie propre pour garder After Effects (irremplaçable) tout en gardant un hôte Linux sans aucun Adobe. Principe : IOMMU (VT-d / AMD-Vi) mappe le GPU directement dans une VM Windows 11, qui en prend le contrôle exclusif avec des perfs quasi-natives (CUDA/NVENC inclus). Resolve, Photoshop, Illustrator y tournent "genuinely native". Heiko Sieger (12 ans de passthrough) y édite du 4K/8K en production.

À savoir avant de se lancer :
- **Confort = 2 GPU** (un iGPU ou petit GPU pour l'hôte, un GPU dédié à la VM). Le single-GPU passthrough est possible mais instable (l'hôte perd tout affichage pendant la session VM).
- **Looking Glass** relaie le framebuffer de la VM vers une fenêtre sur l'hôte à très basse latence, sans second moniteur physique. Viable pour le montage ; potentiellement gênant en étalonnage haute précision.
- **Piège majeur sur Blackwell (RTX 50)** : un *reset bug* confirmé fait que le GPU ne répond plus après l'arrêt de la VM (FLR PCIe qui échoue, GPU qui disparaît de `lspci`, reboot physique requis). **Ce bug touche les guests Windows ET Linux.** Une RTX 4090 (Ada) n'est PAS affectée. **Conséquence : pour du passthrough Adobe stable aujourd'hui, un rig RTX 4090 est un meilleur candidat qu'un RTX 5090, tant que le bug Blackwell n'est pas corrigé.**
- Guest **Windows 11** obligatoire (Photoshop 26.9+ ne démarre pas sur un guest Windows 10).

### 1.4 Ce que font réellement les pros vidéo qui passent à Linux

Trois patterns, aucun ne passe par Wine pour la vidéo :
1. **Full switch natif** : Resolve + Fusion + darktable + Krita/GIMP + Reaper. Choix dominant et croissant, porté par la maturité de Resolve.
2. **Dual-boot** : Linux au quotidien, reboot Windows pour Adobe quand strictement nécessaire. Pragmatique, le filet de sécurité le plus simple.
3. **VM passthrough** : hôte Linux autonome + VM Windows pour les 2-3 tâches Adobe irremplaçables (surtout After Effects).

---

## 2. Sortir de Microsoft (et de Google)

| Besoin | Recommandation | Friction |
|---|---|---|
| Bureautique | **OnlyOffice Desktop** (fidélité .docx native) + **LibreOffice 26.2** (richesse) | Faible |
| Mail / Agenda / Contacts | **Thunderbird** + fournisseur respectueux (**Infomaniak** ou **Mailbox.org**). **NE PAS auto-héberger.** | Faible |
| Sync médias entre machines | **Syncthing** (P2P, zéro serveur) | Faible |
| Cloud / partage externe (option) | **Nextcloud** (pack complet) ou **Seafile** (sync rapide + liens) | Moyenne |
| Visio | **Jitsi Meet** (lien sans compte) | Faible |
| Chat d'équipe (option) | **Element / Matrix** (E2E) si vrai besoin Slack/Teams autonome | Élevée |
| Notes / docs | **Obsidian + Syncthing** (Markdown que tu possèdes) | Faible |
| Navigateur | **LibreWolf** (principal) + **Mullvad Browser** (sensible) | Faible |
| Recherche | **Brave Search** (quotidien) + **SearXNG** auto-hébergé (contrôle total) | Faible |
| Mots de passe | **Vaultwarden** derrière le VPN privé | Faible |
| IA / Copilot | **Open WebUI** au-dessus d'**Ollama** local (Mistral 24B pour le français) | Faible |

Les trois décisions qui comptent :
1. **N'auto-héberge PAS ton mail.** C'est le seul vrai piège. Gmail est passé en novembre 2025 à un statut binaire Pass/Fail (zéro marge), les IP de VPS bon marché sont pré-blacklistées Spamhaus, Outlook bloque sur listing, et c'est 2 à 5 h/mois de maintenance. Un fournisseur respectueux suisse ou allemand (Infomaniak, Mailbox.org) donne 95 % de la autonomie pour 5 % de l'effort. Si on tient à recevoir en auto-hébergé : relayer l'envoi via un service transactionnel (Postmark, SES) pour éviter le problème de réputation IP.
2. **Syncthing est le pilier sync, pas Nextcloud.** Syncthing synchronise des dossiers entre tes machines en P2P direct, sans serveur. Nextcloud est un "Google Workspace à soi" (partage externe, apps) : ne l'ajouter que si on utilise vraiment ses apps annexes.
3. **Pour la bureautique avec des collaborateurs sous Office** : OnlyOffice gagne (OOXML natif, fidélité de rendu supérieure). Aucune suite libre ne gère les macros VBA complexes : à tester si on en dépend.

La question télémétrie (le "pourquoi covert") :
- **Windows 11 Home/Pro : la télémétrie "Required" ne peut PAS être désactivée** (le niveau "off" n'existe que sur Enterprise/Education). Le compte Microsoft est devenu quasi obligatoire (les contournements OOBE\BYPASSNRO ont été fermés en 25H2). Windows Recall (screenshots IA) est opt-in et limité aux PC Copilot+, mais symptomatique.
- **Linux : zéro télémétrie par défaut** sur Debian, Mint, Fedora, Pop!_OS. Ubuntu fait exception (Ubuntu Insights, hardware anonymisé, opt-out en deux clics) : très en dessous de Windows, mais pas "zéro".

---

## 3. Bureau distant covert (remplacer Parsec et AnyDesk)

Parsec ne supporte pas l'hôte Linux. AnyDesk passe par ses serveurs. Les deux sont exclus pour un stack covert.

| Solution | Open-source | Auto-hébergé | NVENC GPU | Hôte Linux headless | Client macOS | Latence | Verdict |
|---|---|---|---|---|---|---|---|
| **Sunshine + Moonlight** | Oui (GPL-3) | Oui (100 % P2P) | **Oui (NVENC, AV1)** | Oui (EDID virtuel ou dummy plug) | Oui | 5-40 ms | **LE choix travail GPU** |
| **RustDesk self-hosted** | Oui (AGPL-3) | Oui (relais hbbs+hbbr) | Non | Oui | Oui | 30-80 ms | Admin / prise de contrôle |
| NoMachine | Non | Oui | Non | Oui | Oui | 10-50 ms | Propriétaire, non auditable |
| Apache Guacamole | Oui | Oui | Non | via VNC/RDP | navigateur | 50-150 ms | Bastion web fallback |

**LA réponse covert à Parsec pour piloter un GPU Linux : Sunshine (hôte) + Moonlight (client).** Open-source intégral, NVENC/AV1 sur RTX 4090/5090, latence type Parsec, 100 % peer-to-peer (aucun serveur tiers). Hôte Linux headless sans écran : soit un dummy plug HDMI à 6 €, soit un display virtuel via EDID firmware (driver NVIDIA 550+). Le tout passe dans un VPN privé. **RustDesk self-hosted** est le complément pour l'admin classique (mais sans NVENC, donc pas pour le travail GPU interactif).

### La couche réseau autonome (le vrai "100 % covert")

Un VPN mesh pratique repose souvent sur un plan de contrôle propriétaire (le serveur de coordination voit les métadonnées de connexion, pas le contenu, qui reste chiffré E2E). Pour être pleinement autonome :

| Solution | Plan de contrôle | Auto-hébergé | NAT traversal | Autonomie |
|---|---|---|---|---|
| VPN mesh grand public (type Tailscale) | hébergé par l'éditeur | client open, serveur non | oui | Partielle |
| **Headscale** | **auto-hébergé** | oui (BSD-3) | oui | Complète |
| **NetBird self-hosted** | **auto-hébergé** | oui (BSD-3) | oui | Complète |
| WireGuard manuel | aucun (statique) | oui | manuel | Complète |

Avis franc : pour la grande majorité des usages, un VPN mesh grand public suffit (trafic chiffré E2E, seul le plan de contrôle voit les métadonnées). Pour être vraiment covert (par principe ou par besoin), basculer le plan de coordination en auto-hébergé : **Headscale** (on garde les mêmes clients, on change juste le serveur de coordination) ou **NetBird** (stack indépendant, interface web moderne).

**Stack remote autonome recommandé** : Sunshine + Moonlight (travail GPU) + RustDesk self-hosted (admin) + réseau WireGuard via Headscale ou NetBird auto-hébergé sur une machine always-on.

---

## 4. Quelle distro ? Ubuntu vs Fedora vs Pop!_OS

C'est le seul vrai arbitrage du poste, et il dépend de la priorité :

- **Ubuntu 24.04 LTS** si la priorité est l'**alignement avec l'outillage NVIDIA d'entreprise** (DGX OS en dérive, containers NGC, AI Workbench, le maximum de tutos et de support). Léger bémol télémétrie (opt-out). C'est le choix de l'étude de migration pour la posture "DGX-like".
- **Fedora Workstation** si la priorité est le **support GPU le plus frais** (kernel le plus récent = meilleur Blackwell/RTX 5090) **et zéro télémétrie**. Drivers NVIDIA via RPM Fusion (2 commandes), CUDA opérationnel.
- **Pop!_OS** (System76) si la priorité est le **plug-and-play NVIDIA** : ISO avec drivers préinstallés, CUDA out-of-the-box, zéro télémétrie, GNOME familier.

À éviter pour un poste GPU créatif : **Qubes OS / Tails / Whonix** (le passthrough PCI pour le GPU est complexe ou impossible, on perd son outil de travail pour une sécurité dont on n'a pas l'usage). C'est de la paranoïa contre-productive ici.

Synthèse : **Ubuntu pour coller au stack NVIDIA pro, Fedora/Pop!_OS pour la fraîcheur kernel + zéro télémétrie.** Sur Blackwell, les trois fonctionnent ; le choix est une question de curseur entre alignement entreprise et autonomie/fraîcheur.

---

## 5. Le verdict covert : trois niveaux de pureté

Choisir en conscience, du plus autonome au plus pragmatique :

1. **100 % open-source** : Fedora/Pop!_OS + Resolve (gratuit) + Blender + GIMP/Krita + darktable + Ardour + Sunshine/Moonlight + Headscale + LibreOffice + Syncthing + Vaultwarden + Open WebUI. On accepte les trous (motion design AE, CMYK pro). Autonomie maximale.
2. **Autonome "pratique"** (recommandé pour un pro) : idem, mais on s'autorise des logiciels propriétaires *natifs et sans abonnement* (Resolve Studio 295 $, Reaper 60 $) et un fournisseur mail respectueux plutôt que l'auto-hébergement. 95 % de la autonomie, 20 % de l'effort.
3. **Transition douce** : hôte Linux autonome + VM Windows à passthrough GPU pour les 2-3 tâches Adobe irremplaçables (surtout After Effects), ou dual-boot. On garde Adobe le temps de s'en sevrer.

**La vérité nette** : on peut bâtir un poste créatif Linux quasi complet en natif aujourd'hui. Les deux seuls points qui forcent encore un compromis sont **After Effects (motion design)** et le **CMYK pro / Generative Fill de Photoshop**. Pour le reste, le natif Linux est mûr, et le bureau distant + le réseau peuvent être rendus 100 % auto-hébergés. La seule chose que Wine apporte vraiment, c'est Photoshop ; pour la vidéo, c'est natif (Resolve) ou VM passthrough, jamais Wine.

---

## Sources

Recherche web de juin 2026, vérification adversariale sur la section Adobe. Références principales : DaVinci Resolve 21 (petapixel, pugetsystems pour les codecs), percée Wine/Photoshop janvier 2026 (phoronix, tomshardware, ghacks), VM passthrough et reset bug Blackwell (looking-glass.io, level1techs, tomshardware), darktable/GIMP/Krita/Inkscape/Scribus/Reaper (sites officiels), stack autonome (privacyguides, itsfoss, syncthing.net, vaultwarden, open-webui), remote/réseau (lizardbyte Sunshine, moonlight-stream, rustdesk.com, headscale, netbird.io).

## Licence

Texte sous CC BY-NC-ND 4.0, extraits de code sous PolyForm Noncommercial 1.0.0. Voir `LICENSE`.
