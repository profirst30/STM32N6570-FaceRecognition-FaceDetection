# Architecture Complète — STM32N6 Face Recognition

Ce document décrit en détail l'architecture de la codebase, composant par composant.
Il peut servir de référence pour comprendre comment chaque pièce du système fonctionne et
comment elles interagissent entre elles.

---

## Table des matières

1. [Vue d'ensemble du système](#1-vue-densemble-du-système)
2. [Matériel cible — STM32N6570-DK](#2-matériel-cible--stm32n6570-dk)
3. [Carte mémoire — Flash externe et RAM](#3-carte-mémoire--flash-externe-et-ram)
4. [Arborescence du projet](#4-arborescence-du-projet)
5. [Chaîne de démarrage (Boot)](#5-chaîne-de-démarrage-boot)
6. [Pipeline principal — Les 6 étapes](#6-pipeline-principal--les-6-étapes)
7. [Détection de visages — CenterFace](#7-détection-de-visages--centerface)
8. [Reconnaissance faciale — MobileFaceNet](#8-reconnaissance-faciale--mobilefacenet)
9. [Banque d'embeddings et similarité cosinus](#9-banque-dembed-dings-et-similarité-cosinus)
10. [Système de vote sur 5 frames](#10-système-de-vote-sur-5-frames)
11. [Cropping et alignement du visage](#11-cropping-et-alignement-du-visage)
12. [Caméra et ISP](#12-caméra-et-isp)
13. [Affichage LCD](#13-affichage-lcd)
14. [Interaction utilisateur — Bouton et LEDs](#14-interaction-utilisateur--bouton-et-leds)
15. [Communication UART — PC Stream](#15-communication-uart--pc-stream)
16. [NPU — Neural Processing Unit](#16-npu--neural-processing-unit)
17. [Initialisation système](#17-initialisation-système)
18. [Build system — Makefile](#18-build-system--makefile)
19. [Scripts de toolchain](#19-scripts-de-toolchain)
20. [Configuration centralisée](#20-configuration-centralisée)
21. [Middlewares et BSP](#21-middlewares-et-bsp)
22. [Schéma de dépendances entre fichiers](#22-schéma-de-dépendances-entre-fichiers)

---

## 1. Vue d'ensemble du système

Le système est un pipeline embarqué temps réel de détection et reconnaissance faciale
qui tourne sur une carte STM32N6570-DK. Il capture des images depuis une caméra,
détecte les visages avec un premier réseau de neurones (CenterFace), puis identifie
chaque visage détecté avec un second réseau (MobileFaceNet).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STM32N6570-DK BOARD                              │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  CAMÉRA  │───>│   ISP    │───>│   NPU    │───>│   LCD    │      │
│  │ (IMX335) │    │ (DCMIPP) │    │(Cortex-  │    │ (800x480)│      │
│  │          │    │          │    │  M55+NPU)│    │          │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│                                       │                             │
│                                       ▼                             │
│                              ┌──────────────┐                       │
│                              │  FLASH EXT.  │                       │
│                              │  (64 MB NOR) │                       │
│                              │  Modèles AI  │                       │
│                              └──────────────┘                       │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                      │
│  │  BOUTON  │    │   LEDs   │    │   UART   │                      │
│  │  USER1   │    │ LED1+LED2│    │ (debug)  │                      │
│  └──────────┘    └──────────┘    └──────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Résumé des flux de données :**

1. La caméra capture une image
2. Le DCMIPP (ISP matériel) produit deux sorties : une pour l'affichage, une pour le réseau de neurones
3. L'image 128×128 passe dans CenterFace → boîtes englobantes des visages
4. Chaque visage est découpé, aligné, redimensionné en 112×112
5. L'image 112×112 passe dans MobileFaceNet → vecteur d'embedding de 128 floats
6. L'embedding est comparé à l'embedding cible par similarité cosinus
7. Le résultat est affiché sur l'écran LCD et les LEDs

---

## 2. Matériel cible — STM32N6570-DK

| Composant | Spécification |
|---|---|
| **Processeur** | ARM Cortex-M55 @ 800 MHz |
| **NPU** | Accélérateur neuronal intégré ST Neural-ART |
| **RAM interne** | AXISRAM1 (1 MB), AXISRAM3-6 (4×448 KB pour NPU) |
| **RAM externe** | PSRAM 16 MB (APS256XX, accès XSPI) |
| **Flash externe** | NOR 64 MB (MX66UW1G45G, accès XSPI) |
| **Caméra** | IMX335 (5MP), VD55G1, ou VD66GY |
| **Écran LCD** | 800×480 pixels, interface LTDC |
| **Interface debug** | SWD + UART via USB ST-Link |
| **Boutons** | USER1 (GPIO) |
| **LEDs** | LED1 (rouge), LED2 (verte) |

---

## 3. Carte mémoire — Flash externe et RAM

### Flash externe (NOR 64 MB, base 0x70000000)

Quatre zones distinctes sont flashées séparément :

```
    Adresse Flash          Contenu                  Taille approx.
┌─────────────────┐
│   0x70000000    │  FSBL (First Stage Boot     ~  64 KB
│                 │  Loader)
├─────────────────┤
│   0x70100000    │  Application firmware        ~ 700 KB
│                 │  (Project_signed.bin)
├─────────────────┤
│   0x71000000    │  Modèle CenterFace           ~ 2.4 MB
│                 │  (face_detection_data.hex)
├─────────────────┤
│   0x72000000    │  Modèle MobileFaceNet        ~ 1.0 MB
│                 │  (face_recognition_data.hex)
├─────────────────┤
│   0x73000000    │  Libre (disponible pour      ~ 47 MB
│   ...           │  stockage persistant, etc.)
│   0x73FFFFFF    │
└─────────────────┘
```

### Mémoire RAM

```
    Adresse                Contenu                   Taille
┌─────────────────┐
│   0x34000400    │  AXISRAM1 — Code applicatif    1023 KB
│                 │  (.text, .data, .bss, stack)
│                 │  Le firmware tourne ici
├─────────────────┤
│   AXISRAM3-6    │  NPU RAM — Mémoire dédiée     4 × 448 KB
│                 │  au Neural Processing Unit
│                 │  (tampons d'inférence)
├─────────────────┤
│   0x91000000    │  PSRAM — Grands buffers        16 MB
│                 │  images, embeddings
│                 │  (section .psram_bss)
└─────────────────┘
```

**Ce qui va en PSRAM (grands buffers) :**
- `nn_rgb[49152]` — Image 128×128×3 pour le réseau de détection
- `fr_rgb[37632]` — Image 112×112×3 pour le réseau de reconnaissance
- `img_buffer[768000]` — Buffer d'affichage 800×480×2 (RGB565)
- `lcd_fg_buffer[2][768000]` — Double buffer LCD foreground

**Ce qui reste en AXISRAM1 (code + données critiques) :**
- Tout le code exécutable
- La pile (16 KB)
- Les variables globales de petite taille
- Le buffer DCMIPP pour la sortie caméra NN

---

## 4. Arborescence du projet

```
STM32N6-FaceRecognition/
│
├── stm32_tools_config.json          Configuration des outils (chemins)
├── stmaic_STM32N6570-DK.conf        Configuration STEdgeAI
├── ARCHITECTURE.md                   Ce document
│
├── input_models/                     MODÈLES AI SOURCES
│   ├── centerface.tflite             Modèle détection (TFLite)
│   ├── mobilefacenet_int8_faces.onnx Modèle reconnaissance (ONNX INT8)
│   ├── mobilefacenet_fp32.onnx       Version FP32 (référence)
│   └── mobilefacenet_int8_random.onnx Version calibrée random
│
├── converted_models/                 MODÈLES CONVERTIS PAR STEDGEAI
│   ├── code/                         Code C généré automatiquement
│   ├── *.onnx                        Modèles ONNX intermédiaires
│   └── *_c_info.json                 Métadonnées des modèles
│
├── scripts/                          SCRIPTS D'OUTILLAGE
│   ├── compile_model.sh              Conversion modèle → code C + hex
│   ├── compile_all_models.sh         Conversion de tous les modèles
│   ├── sign_binary.sh                Signature du firmware
│   ├── flash_firmware.sh             Flashage des 4 composants
│   ├── face_detection.mpool          Pool mémoire détection
│   ├── face_detection_config.json    Config STEdgeAI détection
│   ├── face_recognition.mpool        Pool mémoire reconnaissance
│   └── face_recognition_config.json  Config STEdgeAI reconnaissance
│
├── st_ai_ws/                         WORKSPACE STEDGEAI (artefacts)
│   ├── neural_art__face_detection/   Sortie compilation détection
│   └── neural_art__face_recognition/ Sortie compilation reconnaissance
│
└── embedded/                         PROJET EMBARQUÉ
    ├── Makefile                      Système de build
    ├── STM32CubeIDE/                 Projet IDE + linker script
    │   └── STM32N657xx.ld            Script de lien mémoire
    ├── Binary/                       Binaires finaux flashés
    │   ├── ai_fsbl.hex               Bootloader
    │   ├── Project_signed.bin        Firmware signé (binaire)
    │   ├── Project_signed.hex        Firmware signé (hex)
    │   ├── face_detection_data.hex   Données modèle détection
    │   └── face_recognition_data.hex Données modèle reconnaissance
    │
    ├── Inc/                          FICHIERS D'EN-TÊTE
    │   ├── app_config.h              Configuration globale
    │   ├── app_constants.h           Constantes (seuils, tailles)
    │   ├── app_config_manager.h      Gestionnaire de configuration
    │   ├── app_neural_network.h      Structures réseaux neuronaux
    │   ├── app_frame_processing.h    Structures pipeline
    │   ├── target_embedding.h        API banque d'embeddings
    │   ├── crop_img.h                API traitement d'image
    │   ├── face_utils.h              Similarité cosinus
    │   ├── display_utils.h           API affichage LCD
    │   ├── nn_runner.h               Exécuteur NN synchrone
    │   ├── app_cam.h                 API caméra
    │   ├── enhanced_pc_stream.h      Protocole UART
    │   ├── memory_pool.h             Gestionnaire mémoire
    │   └── ...                       (BSP, HAL, ISP configs)
    │
    ├── Src/                          CODE SOURCE APPLICATIF
    │   ├── main.c                    Point d'entrée + pipeline
    │   ├── nn_runner.c               Exécution réseau sur NPU
    │   ├── app_postprocess.c         Post-traitement détection
    │   ├── app_cam.c                 Gestion caméra DCMIPP
    │   ├── crop_img.c                Découpe/alignement visage
    │   ├── face_utils.c              Similarité cosinus
    │   ├── target_embedding.c        Banque d'embeddings
    │   ├── display_utils.c           Rendu LCD
    │   ├── app_system.c              Initialisation hardware
    │   ├── system_utils.c            Clocks, NPU, sécurité
    │   ├── enhanced_pc_stream.c      Communication UART
    │   ├── app_config_manager.c      Gestion configuration
    │   ├── img_buffer.c              Buffer image LCD
    │   └── ...
    │
    ├── Models/                       CODE C GÉNÉRÉ DES MODÈLES
    │   ├── face_detection.c          Poids + graphe détection
    │   ├── face_detection.h          Interface détection
    │   ├── face_recognition.c        Poids + graphe reconnaissance
    │   └── face_recognition.h        Interface reconnaissance
    │
    ├── Middlewares/                   COUCHES LOGICIELLES
    │   ├── AI_Runtime/               Runtime ATON (LL_ATON)
    │   ├── Camera_Middleware/         Abstraction caméra (CMW)
    │   └── lib_vision_models_pp/     Post-processing modèles vision
    │
    └── STM32Cube_FW_N6/              FIRMWARE ST
        └── Drivers/
            ├── CMSIS/                Core ARM + NN
            ├── STM32N6xx_HAL_Driver/ HAL ST complet
            └── BSP/STM32N6570-DK/    Board Support Package
```

---

## 5. Chaîne de démarrage (Boot)

Au power-on, le STM32N6 suit cette séquence :

```
 POWER ON
    │
    ▼
┌──────────────────────────┐
│  ROM interne STM32N6     │  Le microcontrôleur démarre depuis
│  (Boot ROM ST)           │  sa ROM interne et cherche le FSBL
└───────────┬──────────────┘  en flash externe
            │
            ▼
┌──────────────────────────┐
│  FSBL @ 0x70000000       │  Le First Stage Boot Loader :
│  (ai_fsbl.hex)           │  - Configure les horloges de base
│                          │  - Initialise la mémoire externe
│                          │  - Vérifie la signature de l'appli
│                          │  - Charge l'appli en AXISRAM1
│                          │  - Saute à l'application
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│  Application             │  Le firmware applicatif :
│  @ 0x70100000            │  - app_init() : init hardware
│  (Project_signed.hex)    │  - app_main_loop() : boucle infinie
│                          │  - Accède aux modèles AI
│                          │    en flash via memory-mapping
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│  Modèles AI en flash     │  Les poids des réseaux sont lus
│  Détection @ 0x71000000  │  directement depuis la flash
│  Reco.    @ 0x72000000   │  via le memory-mapping XSPI
└──────────────────────────┘  (pas de copie en RAM)
```

**Pourquoi signer le firmware ?**
Le STM32N6 exige un firmware signé pour démarrer. Le SigningTool ajoute
un en-tête SSFI (Secure Software Firmware Image) que le FSBL vérifie
avant d'exécuter l'application.

---

## 6. Pipeline principal — Les 6 étapes

La fonction `app_main_loop()` dans `main.c` exécute une boucle infinie
avec 6 étapes clairement séparées :

```
 ═══════════════════════════════════════════════════════════════
  BOUCLE PRINCIPALE (exécutée à chaque frame, ~6 FPS)
 ═══════════════════════════════════════════════════════════════
                          │
   ┌──────────────────────▼──────────────────────────┐
   │  ÉTAPE 1 : Capture & Prétraitement              │
   │  ─────────────────────────────────               │
   │  • Capturer une frame de la caméra               │
   │  • Convertir RGB (HWC) → Float CHW               │
   │  • Copier dans le buffer d'entrée du NPU        │
   │  • Nettoyer le cache (SCB_CleanInvalidateDCache)  │
   └──────────────────────┬──────────────────────────┘
                          │
   ┌──────────────────────▼──────────────────────────┐
   │  ÉTAPE 2 : Détection de visages                  │
   │  ────────────────────────────────                 │
   │  • Exécuter CenterFace sur le NPU               │
   │  • RunNetworkSync() bloque jusqu'à la fin        │
   │  • Le NPU produit 4 tenseurs de sortie          │
   └──────────────────────┬──────────────────────────┘
                          │
   ┌──────────────────────▼──────────────────────────┐
   │  ÉTAPE 3 : Post-traitement                       │
   │  ───────────────────────                          │
   │  • Décoder les 4 sorties du réseau :             │
   │    scale, landmarks, heatmap, offset              │
   │  • Appliquer le seuil de confiance (0.5)         │
   │  • NMS (Non-Maximum Suppression, IoU=0.3)        │
   │  • Résultat : liste de boîtes englobantes        │
   └──────────────────────┬──────────────────────────┘
                          │
   ┌──────────────────────▼──────────────────────────┐
   │  ÉTAPE 4 : Reconnaissance faciale                │
   │  ────────────────────────────────                 │
   │  Pour CHAQUE visage détecté :                    │
   │  • Convertir les coordonnées normalisées→pixels  │
   │  • Découper et aligner le visage (112×112)       │
   │  • Convertir en float normalisé [-1, +1]         │
   │  • Exécuter MobileFaceNet sur le NPU            │
   │  • Calculer la similarité cosinus avec la cible  │
   │  • Mettre à jour l'historique de vote            │
   └──────────────────────┬──────────────────────────┘
                          │
   ┌──────────────────────▼──────────────────────────┐
   │  ÉTAPE 5 : Mise à jour du système               │
   │  ──────────────────────────────                   │
   │  • Allumer/éteindre les LEDs selon le résultat   │
   │  • Lire le bouton USER1 (ajout/reset embedding)  │
   │  • Envoyer un heartbeat UART                     │
   └──────────────────────┬──────────────────────────┘
                          │
   ┌──────────────────────▼──────────────────────────┐
   │  ÉTAPE 6 : Sortie & Métriques                    │
   │  ─────────────────────────────                    │
   │  • Calculer les FPS                              │
   │  • Dessiner les boîtes et scores sur le LCD      │
   │  • Afficher le visage découpé en miniature       │
   │  • Afficher FPS, nombre d'embeddings, boot time  │
   │  • Nettoyer les buffers de sortie NN             │
   └──────────────────────┬──────────────────────────┘
                          │
                          └─── Retour à l'Étape 1
```

---

## 7. Détection de visages — CenterFace

### Modèle

| Propriété | Valeur |
|---|---|
| Architecture | CenterFace (anchor-free detector) |
| Fichier source | `input_models/centerface.tflite` |
| Entrée | 128×128×3 pixels, float32, format CHW |
| Sorties | 4 tenseurs (scale, landmarks, heatmap, offset) |
| Adresse flash | `0x71000000` |

### Comment ça fonctionne

CenterFace est un détecteur "anchor-free", ce qui veut dire qu'il ne
prédéfinit pas de boîtes de taille fixe. Au lieu de cela, il prédit :

```
  Image 128×128
       │
       ▼
  ┌────────────┐
  │ CenterFace │
  │  (NPU)     │
  └─┬──┬──┬──┬─┘
    │  │  │  │
    ▼  ▼  ▼  ▼
 Sortie 0: scale    ──> Taille des visages (largeur, hauteur)
 Sortie 1: landmarks ──> Position des 5 points clés du visage
 Sortie 2: heatmap   ──> Probabilité de présence d'un centre de visage
 Sortie 3: offset    ──> Ajustement fin de la position du centre
```

Les **5 landmarks** (points clés) sont :
1. Œil gauche
2. Œil droit
3. Nez
4. Coin gauche de la bouche
5. Coin droit de la bouche

Seuls les 2 premiers (yeux) sont utilisés pour l'alignement du visage
lors du cropping.

### Post-traitement

Le post-traitement est effectué par la bibliothèque `lib_vision_models_pp`.
L'application appelle `app_postprocess_run()` qui :

1. Prend les 4 tenseurs de sortie
2. Décode la heatmap en positions de centres de visages
3. Affine les positions avec les offsets
4. Calcule les boîtes englobantes avec les scales
5. Applique le seuil de confiance (0.5 par défaut)
6. Applique la NMS (Non-Maximum Suppression) avec un seuil IoU de 0.3
7. Retourne jusqu'à 10 boîtes englobantes avec leurs landmarks

---

## 8. Reconnaissance faciale — MobileFaceNet

### Modèle

| Propriété | Valeur |
|---|---|
| Architecture | MobileFaceNet (basé sur MobileNetV2) |
| Fichier source | `input_models/mobilefacenet_int8_faces.onnx` |
| Quantization | INT8 (calibré sur des visages réels) |
| Entrée | 112×112×3 pixels, float32 normalisé [-1, +1], format CHW |
| Sortie | Vecteur de 128 floats (embedding) |
| Adresse flash | `0x72000000` |

### Comment ça fonctionne

MobileFaceNet transforme une image de visage en un vecteur de 128
dimensions appelé "embedding". Ce vecteur capture l'identité du visage :
deux photos de la même personne produisent des embeddings proches,
deux personnes différentes produisent des embeddings éloignés.

```
  Image visage 112×112
         │
         │  Normalisation : pixel / 127.5 - 1.0
         │  (chaque pixel passe de [0,255] à [-1,+1])
         ▼
  ┌───────────────┐
  │ MobileFaceNet  │
  │   (NPU)       │
  └───────┬───────┘
          │
          ▼
  [0.12, -0.45, 0.78, ..., 0.33]   ← Vecteur de 128 floats
                                       C'est "l'identité numérique"
                                       du visage
```

### Initialisation paresseuse (lazy loading)

Le réseau de reconnaissance n'est PAS chargé au démarrage.
Il est initialisé la première fois qu'un visage est détecté.
Cela réduit le temps de boot d'environ 200 ms.

---

## 9. Banque d'embeddings et similarité cosinus

### Banque d'embeddings

Le fichier `target_embedding.c` gère une banque de jusqu'à 10 embeddings
de référence. Quand l'utilisateur appuie sur le bouton USER1, l'embedding
du visage actuellement visible est ajouté à cette banque.

```
  Banque d'embeddings (en RAM, max 10)
 ┌─────────────────────────────────────┐
 │ Embedding 0: [0.12, -0.45, ...]    │ ← Photo 1 de la personne
 │ Embedding 1: [0.14, -0.42, ...]    │ ← Photo 2 (angle différent)
 │ Embedding 2: [0.11, -0.47, ...]    │ ← Photo 3 (éclairage différent)
 │ ...                                 │
 │ Embedding 9: (vide)                 │
 └─────────────────┬───────────────────┘
                   │
                   │  Moyenne + normalisation L2
                   ▼
 ┌─────────────────────────────────────┐
 │ Target embedding (embedding moyen)  │
 │ [0.123, -0.447, 0.781, ...]        │
 └─────────────────────────────────────┘
```

**Processus de calcul du target :**
1. On fait la moyenne élément par élément de tous les embeddings
2. On normalise le résultat par sa norme L2 (pour que la longueur = 1)

Plus on ajoute d'embeddings (angles, éclairages variés), plus la
reconnaissance est robuste.

### Similarité cosinus

La comparaison entre deux embeddings utilise la similarité cosinus,
calculée dans `face_utils.c` :

```
                        A · B
  similarité(A, B) = ─────────────
                      ||A|| × ||B||

  Où :
  • A · B    = produit scalaire des deux vecteurs
  • ||A||    = norme euclidienne de A
  • ||B||    = norme euclidienne de B
  • Résultat = valeur entre -1.0 et +1.0
```

| Score | Signification |
|---|---|
| > 0.55 | Même personne (seuil par défaut) |
| 0.3 – 0.55 | Incertain |
| < 0.3 | Personnes différentes |

Le seuil de 0.55 est défini par `FACE_SIMILARITY_THRESHOLD` dans
`app_constants.h`.

---

## 10. Système de vote sur 5 frames

Pour éviter les faux positifs ponctuels, le système ne se base pas
sur une seule frame pour décider si la personne cible est présente.
Il utilise un vote sur les 5 dernières frames :

```
  Frame N-4   Frame N-3   Frame N-2   Frame N-1   Frame N
  ┌───┐       ┌───┐       ┌───┐       ┌───┐       ┌───┐
  │ ✓ │       │ ✗ │       │ ✓ │       │ ✓ │       │ ✓ │
  └───┘       └───┘       └───┘       └───┘       └───┘
    │           │           │           │           │
    └───────────┴───────────┴───────────┴───────────┘
                            │
                     Votes positifs : 4/5
                     Seuil : 3/5
                     ───────────────
                     Résultat : CIBLE DÉTECTÉE ✓
```

Le buffer circulaire `target_detection_history[5]` stocke le résultat
de chaque frame. Si au moins 3 des 5 dernières frames ont trouvé la
cible au-dessus du seuil de similarité, `target_detected` passe à `true`.

---

## 11. Cropping et alignement du visage

Le fichier `crop_img.c` contient les fonctions de traitement d'image
les plus critiques du pipeline. Quand un visage est détecté, il faut
le découper et l'aligner avant de le passer au réseau de reconnaissance.

### Étapes de l'alignement

```
  Image caméra (480×480)                      Image alignée (112×112)
 ┌──────────────────────────┐               ┌────────────────┐
 │                          │               │                │
 │     ┌──────────┐         │               │   ┌────────┐   │
 │     │  ◉    ◉  │←visage  │   Rotation   │   │ ◉  ◉  │   │
 │     │    ▲     │ penché   │ ──────────>  │   │   ▲   │   │
 │     │  ╰───╯   │         │  Crop +      │   │ ╰───╯ │   │
 │     └──────────┘         │  Redim.      │   └────────┘   │
 │                          │               │                │
 └──────────────────────────┘               └────────────────┘
```

**Comment fonctionne l'alignement :**

1. **Calcul de l'angle de rotation** : On utilise les positions des deux yeux
   (landmarks 0 et 1) pour calculer l'angle entre la ligne des yeux et
   l'horizontale.

2. **Rotation inverse** : Pour chaque pixel de l'image de sortie 112×112,
   on calcule d'où il vient dans l'image source en appliquant la rotation
   inverse autour du centre du visage.

3. **Clamping** : Les coordonnées sources sont bornées aux limites de
   l'image pour éviter les accès mémoire hors limites.

4. **Padding** : La boîte englobante est élargie de 20% (`FACE_BBOX_PADDING_FACTOR = 1.2`)
   pour inclure le contour du visage.

### Deux variantes de crop

| Fonction | Source | Destination | Usage |
|---|---|---|---|
| `img_crop_align` | RGB888 | RGB888 | Mode PC stream |
| `img_crop_align565_to_888` | RGB565 | RGB888 | Mode caméra (conversion couleur en même temps) |

### Conversion de format

Deux fonctions convertissent les pixels de format HWC (Height-Width-Channel,
le format natif de la caméra) en CHW (Channel-Height-Width, le format
requis par les réseaux) :

| Fonction | Normalisation | Réseau |
|---|---|---|
| `img_rgb_to_chw_float` | Aucune (0–255 → float) | CenterFace |
| `img_rgb_to_chw_float_norm` | pixel/127.5 - 1.0 (→ [-1,+1]) | MobileFaceNet |

---

## 12. Caméra et ISP

### Architecture matérielle

Le fichier `app_cam.c` configure le sous-système caméra du STM32N6
via le Camera Middleware (CMW).

```
  Capteur caméra (IMX335, 5MP)
         │
         │  Interface CSI (MIPI)
         ▼
  ┌────────────────────────┐
  │        DCMIPP           │  Digital Camera Interface
  │  (Image Signal          │  & Pixel Pipeline
  │   Processor matériel)   │
  │                         │
  │  ┌─────────┐ ┌────────┐│
  │  │ PIPE 1  │ │ PIPE 2 ││
  │  │Affichage│ │ Réseau ││
  │  │ RGB565  │ │ RGB888 ││
  │  │ 480×480 │ │128×128 ││
  │  └────┬────┘ └───┬────┘│
  └───────┼──────────┼─────┘
          │          │
          ▼          ▼
     img_buffer    nn_rgb
     (LCD)         (NN input)
```

**Deux pipes DCMIPP parallèles :**

- **PIPE 1 (Display)** : Redimensionne l'image caméra en 480×480 RGB565
  et l'écrit directement dans `img_buffer` pour l'affichage LCD.
  Tourne en mode continu.

- **PIPE 2 (Neural Network)** : Redimensionne l'image caméra en 128×128
  RGB888 et l'écrit dans `nn_rgb` pour le réseau de détection.
  Tourne en mode snapshot (une capture par frame).

### Mode Aspect Ratio

Trois modes sont disponibles (configurés dans `app_config.h`) :
- **CROP** : Coupe l'image pour garder le ratio d'aspect (par défaut)
- **FIT** : Étire l'image pour remplir le rectangle
- **FULLSCREEN** : Mode plein écran

---

## 13. Affichage LCD

Le fichier `display_utils.c` gère l'écran LCD 800×480 avec deux
couches matérielles LTDC :

```
  ┌──────────────────────── LCD 800×480 ────────────────────────┐
  │                                                              │
  │   ┌──────── COUCHE 1 (Background) ────────────┐             │
  │   │                                            │             │
  │   │  Image caméra en direct                    │             │
  │   │  (img_buffer, RGB565)                      │             │
  │   │                                            │             │
  │   │   ┌── COUCHE 2 (Foreground, transparente) ─┤             │
  │   │   │                                        │             │
  │   │   │  Boîtes englobantes (vert/rouge)       │             │
  │   │   │  Scores de similarité                  │             │
  │   │   │  Zone de crop alignée (cyan)           │             │
  │   │   │  Landmarks (points rouges)             │             │
  │   │   │                                        │             │
  │   │   │  ┌─────────┐                           │             │
  │   │   │  │ Miniature│ Visage découpé 112×112   │             │
  │   │   │  │  aligné  │ affiché en bas à droite  │             │
  │   │   │  └─────────┘                           │             │
  │   │   │                                        │             │
  │   │   │  FPS: 6                                │             │
  │   │   │  Embeddings: 3/10                      │             │
  │   │   │  Boot time: 219ms                      │             │
  │   │   └────────────────────────────────────────┘             │
  │   └────────────────────────────────────────────┘             │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

**Code couleur des boîtes englobantes :**
- **Vert** : Visage détecté, similarité ≥ 70% (probablement la cible)
- **Rouge** : Visage détecté, similarité < 70% (personne inconnue)
- **Cyan** : Rectangle de la zone de crop alignée (rotation visible)

Le foreground utilise un **double buffer** (`lcd_fg_buffer[2]`) pour
éviter le tearing : pendant qu'on dessine dans un buffer, le LTDC
affiche l'autre.

---

## 14. Interaction utilisateur — Bouton et LEDs

### Bouton USER1

Le bouton physique USER1 sur la carte a deux actions :

```
  ┌────────────────────────────────────────────────┐
  │                BOUTON USER1                     │
  │                                                 │
  │  Appui court (< 1 seconde) :                   │
  │  ────────────────────────────                   │
  │  → Ajoute l'embedding du visage actuellement    │
  │    visible à la banque (max 10)                 │
  │  → Le compteur "Embeddings: N/10" augmente      │
  │                                                 │
  │  Appui long (≥ 1 seconde) :                    │
  │  ───────────────────────────                    │
  │  → Efface TOUTE la banque d'embeddings          │
  │  → Le compteur repasse à "Embeddings: 0/10"     │
  │  → Aucun visage n'est reconnu ensuite           │
  └────────────────────────────────────────────────┘
```

**Mécanique interne :**
La fonction `handle_user_button()` dans `main.c` détecte l'appui et
le relâchement du bouton. Elle mesure la durée entre les deux pour
distinguer court/long.

### LEDs

```
  ┌──────────────────────────────────────┐
  │  LED1 (rouge)  │  LED2 (verte)       │  Signification
  ├────────────────┼─────────────────────┤
  │    OFF         │    OFF              │  Aucun visage détecté
  │    ON          │    OFF              │  Visage détecté, non reconnu
  │    OFF         │    ON               │  Cible reconnue (3+/5 frames)
  │    Clignotant  │    OFF              │  Erreur système
  └────────────────┴─────────────────────┘
```

Un **timeout d'une seconde** maintient la LED verte allumée après la
dernière reconnaissance positive, pour éviter un clignotement rapide
quand le visage bouge brièvement hors champ.

---

## 15. Communication UART — PC Stream

Le fichier `enhanced_pc_stream.c` implémente un protocole de communication
série avec un PC (optionnel, désactivé par défaut).

### Structure d'un paquet

```
  ┌─────┬──────────────┬──────────┬─────────────────┬──────────┐
  │ SOF │ Payload Size │ Checksum │     Payload      │  CRC32   │
  │0xAA │   2 bytes    │  1 byte  │   N bytes        │ 4 bytes  │
  └─────┴──────────────┴──────────┴─────────────────┴──────────┘
  │◄──── Header (4 bytes) ────────►│◄── Données ────►│◄─ Véri ─►│
```

### Types de messages

| Code | Type | Contenu |
|---|---|---|
| 0x01 | FRAME_DATA | Image brute (miniature grayscale) |
| 0x02 | DETECTION_RESULTS | Boîtes englobantes et scores |
| 0x03 | EMBEDDING_DATA | Vecteur d'embedding (128 floats) |
| 0x04 | PERFORMANCE_METRICS | FPS, temps d'inférence |
| 0x05 | HEARTBEAT | Signal de vie périodique |

---

## 16. NPU — Neural Processing Unit

### Architecture du NPU ST Neural-ART

Le STM32N6 intègre un NPU dédié qui accélère les opérations de réseaux
de neurones (convolutions, pooling, activations). Le runtime ATON
(Artificial Tensor Operations on NPU) gère l'exécution.

```
  Application (Cortex-M55)
         │
         │  LL_ATON API
         ▼
  ┌──────────────────────────┐
  │     Runtime ATON          │
  │  (ll_aton_runtime)       │
  │                          │
  │  - Charge le graphe      │
  │  - Planifie les epochs   │
  │  - Gère le cache NPU     │
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │     NPU Hardware         │
  │                          │
  │  AXISRAM3 ─┐             │
  │  AXISRAM4 ─┤ Scratch     │
  │  AXISRAM5 ─┤ memory      │
  │  AXISRAM6 ─┘ (1.75 MB)   │
  │                          │
  │  Lit les poids depuis    │
  │  la flash (memory-mapped)│
  └──────────────────────────┘
```

### Exécution synchrone

La fonction `RunNetworkSync()` dans `nn_runner.c` est très simple :
elle initialise le réseau, puis exécute les "epoch blocks" un par un
jusqu'à ce que l'inférence soit terminée. Le processeur attend (WFE)
entre chaque epoch, permettant au NPU de travailler.

### Cycle de vie d'un réseau

```
  LL_ATON_RT_Init_Network()       ← Prépare le réseau
         │
         ├──> LL_ATON_RT_RunEpochBlock()  ← Exécute un bloc
         │         │
         │    LL_ATON_OSAL_WFE()  ← CPU attend le NPU
         │         │
         ├──> LL_ATON_RT_RunEpochBlock()  ← Bloc suivant
         │    ...
         │
  LL_ATON_RT_DONE                 ← Inférence terminée
         │
  LL_ATON_RT_DeInit_Network()     ← Libère les ressources
```

**Important** : Le réseau de reconnaissance est déinitialisé après
chaque utilisation (`DeInit_Network`) car les deux réseaux partagent
les mêmes zones de mémoire NPU.

---

## 17. Initialisation système

La fonction `App_SystemInit()` dans `app_system.c` orchestre toute
l'initialisation hardware au démarrage :

```
  App_SystemInit()
       │
       ├── 1. Activer le cache d'instructions (ICache)
       ├── 2. Activer le cache de données (DCache)
       ├── 3. Configurer les horloges système
       │       └── PLL1 = 800 MHz (CPU)
       │       └── PLL2 = périphériques
       │       └── PLL3 = ISP
       │       └── PLL4 = NPU/LTDC
       │
       ├── 4. Activer la NPU et ses 4 bancs SRAM
       │       └── AXISRAM3, 4, 5, 6
       │
       ├── 5. Initialiser la mémoire externe
       │       ├── PSRAM (XSPI RAM, memory-mapped)
       │       └── Flash NOR (XSPI NOR, memory-mapped)
       │
       ├── 6. Programmer les fusibles (Fuse_Programming)
       ├── 7. Configurer le cache NPU
       ├── 8. Configurer la sécurité (RISC, RIMC)
       │       └── Donner accès au NPU, DMA2D, DCMIPP, LTDC
       └── 9. Configurer les horloges en mode sleep
```

Ensuite, `app_init()` dans `main.c` enchaîne :

```
  app_init()
       │
       ├── 1. Initialiser le configuration manager
       ├── 2. App_SystemInit()  (ci-dessus)
       ├── 3. Initialiser le runtime ATON
       ├── 4. Initialiser la banque d'embeddings (vide)
       ├── 5. Initialiser LEDs et bouton
       ├── 6. Charger le réseau de détection
       │       └── (reconnaissance = lazy, chargée plus tard)
       ├── 7. Initialiser le PC stream (UART)
       └── 8. Initialiser le post-traitement
```

---

## 18. Build system — Makefile

Le Makefile dans `embedded/` compile l'ensemble du firmware
avec la chaîne ARM GCC.

### Commandes principales

| Commande | Action |
|---|---|
| `make` | Compile tout le projet |
| `make -j8` | Compile en parallèle (8 threads) |
| `make clean` | Supprime tous les fichiers compilés |

### Processus de compilation

```
  Sources C (Src/*.c, Models/*.c, Middlewares/*.c, HAL/*.c)
       │
       │  arm-none-eabi-gcc -mcpu=cortex-m55 -mfpu=fpv5-sp-d16
       │                    -Os -g3
       ▼
  Fichiers objet (build/*.o)
       │
       │  arm-none-eabi-gcc -T STM32N657xx.ld
       ▼
  build/Project.elf
       │
       ├──> arm-none-eabi-objcopy -O binary → build/Project.bin
       └──> arm-none-eabi-size               → affiche taille mémoire
```

### Flags de compilation importants

| Flag | Rôle |
|---|---|
| `-mcpu=cortex-m55` | Cible le processeur ARM Cortex-M55 |
| `-mfpu=fpv5-sp-d16` | Active le FPU simple précision |
| `-mfloat-abi=hard` | Utilise les registres FPU pour les floats |
| `-Os` | Optimise pour la taille |
| `-DSTM32N657xx` | Définit le microcontrôleur cible |

---

## 19. Scripts de toolchain

### `compile_model.sh` — Conversion de modèle AI

Ce script convertit un modèle ONNX ou TFLite en code C exécutable
sur le NPU du STM32N6.

```
  Entrée:                      Sortie:
  centerface.tflite      ──>   face_detection.c / .h
                               face_detection_ecblobs.h
                               face_detection_data.hex
                               face_detection.mpool

  mobilefacenet.onnx     ──>   face_recognition.c / .h
                               face_recognition_ecblobs.h
                               face_recognition_data.hex
                               face_recognition.mpool
```

Le script utilise l'outil `stedgeai` (STEdgeAI) qui :
1. Analyse le graphe du modèle
2. Optimise les opérations pour le NPU
3. Génère le code C des fonctions d'inférence
4. Génère les blobs binaires des poids (flashés en mémoire externe)
5. Génère les fichiers `.mpool` de configuration mémoire

### `sign_binary.sh` — Signature du firmware

Le STM32N6 exige un firmware signé. Ce script appelle le
`STM32_SigningTool_CLI` pour ajouter un en-tête SSFI :

```
  Project.bin  ──>  STM32_SigningTool_CLI  ──>  Project_signed.bin
                    (mode SSFI, no-key)         Project_signed.hex
```

### `flash_firmware.sh` — Flashage sur la carte

Ce script flashe les 4 composants via `STM32_Programmer_CLI` :

```
  Composant 1 : ai_fsbl.hex              @ 0x70000000
  Composant 2 : Project_signed.hex       @ 0x70100000 (auto)
  Composant 3 : face_detection_data.hex  @ 0x71000000
  Composant 4 : face_recognition_data.hex@ 0x72000000
```

Utilisation : `./flash_firmware.sh all` pour tout flasher d'un coup.

---

## 20. Configuration centralisée

### `stm32_tools_config.json` — Chemins des outils

Ce fichier JSON à la racine contient les chemins de tous les outils
nécessaires (STM32CubeIDE, STEdgeAI, Programmer, SigningTool, ARM GCC)
ainsi que la carte mémoire des modèles.

### `app_config.h` — Configuration du firmware

Paramètres de compilation statiques :

| Paramètre | Valeur | Description |
|---|---|---|
| `NN_WIDTH` / `NN_HEIGHT` | 128×128 | Taille d'entrée de CenterFace |
| `LCD_FG_WIDTH` / `LCD_FG_HEIGHT` | 800×480 | Résolution LCD |
| `NB_CLASSES` | 1 | Une seule classe : "face" |
| `COLOR_MODE` | RGB | Ordre des canaux couleur |
| `ASPECT_RATIO_MODE` | CROP | Mode de ratio d'aspect caméra |
| `AI_PD_MODEL_PP_CONF_THRESHOLD` | 0.5 | Seuil confiance détection |
| `AI_PD_MODEL_PP_IOU_THRESHOLD` | 0.3 | Seuil NMS |
| `AI_PD_MODEL_PP_MAX_BOXES_LIMIT` | 10 | Max visages par frame |
| `ENABLE_LCD_DISPLAY` | défini | Active l'affichage LCD |
| `INPUT_SRC_MODE` | CAMERA | Source = caméra |

### `app_constants.h` — Constantes applicatives

| Constante | Valeur | Rôle |
|---|---|---|
| `FACE_SIMILARITY_THRESHOLD` | 0.55 | Seuil reconnaissance |
| `FACE_DETECTION_CONFIDENCE_THRESHOLD` | 0.7 | Seuil détection min pour lancer la reconnaissance |
| `FACE_BBOX_PADDING_FACTOR` | 1.2 | Marge autour du visage (+20%) |
| `EMBEDDING_SIZE` | 128 | Dimension du vecteur d'embedding |
| `EMBEDDING_BANK_SIZE` | 10 | Nombre max d'embeddings stockés |
| `BUTTON_LONG_PRESS_DURATION_MS` | 1000 | Durée appui long (ms) |
| `FACE_REVERIFY_INTERVAL_MS` | 1000 | Intervalle re-vérification |

---

## 21. Middlewares et BSP

### AI_Runtime (ATON)

Le runtime d'inférence pour le NPU ST Neural-ART. Fournit l'API
`LL_ATON_RT_*` pour charger et exécuter les réseaux de neurones.
Contient aussi la gestion du cache NPU et la configuration matérielle.

### Camera_Middleware (CMW)

Abstraction de la chaîne d'acquisition caméra. Gère :
- L'initialisation du capteur (IMX335, VD55G1, VD66GY)
- La configuration des pipes DCMIPP (résolution, format, ratio)
- L'ISP (Image Signal Processor) pour le traitement d'image matériel
- Les callbacks de fin de frame

### lib_vision_models_pp

Bibliothèque de post-traitement pour modèles de vision. Fournit les
décodeurs spécifiques à chaque architecture de détection :
- CenterNet/CenterFace (utilisé ici)
- YOLOv2, YOLOv5, YOLOv8
- SSD, ST-YOLOX

### BSP (Board Support Package)

Drivers spécifiques à la carte STM32N6570-DK :
- `stm32n6570_discovery.c` : Init générale, boutons, LEDs, COM
- `stm32n6570_discovery_lcd.c` : Contrôle LTDC + écran
- `stm32n6570_discovery_xspi.c` : Flash NOR + PSRAM
- `stm32n6570_discovery_bus.c` : Bus I2C pour la caméra
- Drivers composants : APS256XX (PSRAM), MX66UW1G45G (Flash NOR)

### HAL (Hardware Abstraction Layer)

Les drivers HAL ST pour le STM32N6xx couvrent tous les périphériques
utilisés : GPIO, I2C, LTDC, DCMIPP, XSPI, UART, RCC, PWR, DMA2D,
RAMCFG, BSEC, CRC, CACHEAXI, EXTI, RIF.

---

## 22. Schéma de dépendances entre fichiers

```
                         main.c
                           │
           ┌───────────────┼───────────────────┐
           │               │                   │
           ▼               ▼                   ▼
      app_system.c    nn_runner.c        display_utils.c
           │               │                   │
           ▼               ▼                   ▼
     system_utils.c   ll_aton_runtime     stm32_lcd_ex.c
           │          (AI_Runtime)        BSP LCD driver
           ▼               │
     HAL drivers           ▼
     (RCC, PWR,      face_detection.c ←── Models/
      XSPI, ...)     face_recognition.c    (code généré par
                           │                 STEdgeAI)
                           │
           ┌───────────────┼────────────────┐
           │               │                │
           ▼               ▼                ▼
    app_postprocess.c  crop_img.c    target_embedding.c
           │               │                │
           ▼               │                ▼
    lib_vision_models_pp   │         face_utils.c
    (CenterFace decoder)   │         (cosine similarity)
                           │
                           ▼
                      app_cam.c
                           │
                           ▼
                   Camera_Middleware
                   (CMW + ISP + DCMIPP)
                           │
                           ▼
                    Capteur caméra
                    (IMX335 / VD55G1)
```

### Flux de données entre les fichiers

```
  app_cam.c ──[RGB565 480×480]──> img_buffer.c ──> display_utils.c
      │
      └──[RGB888 128×128]──> nn_rgb ──> main.c (étape 1)
                                            │
                                    crop_img.c (rgb→chw_float)
                                            │
                                    main.c (étape 2) ──> nn_runner.c
                                            │                  │
                                    app_postprocess.c ◄────────┘
                                            │
                                    main.c (étape 4)
                                            │
                              ┌─────────────┼──────────────┐
                              ▼             ▼              ▼
                        crop_img.c    nn_runner.c   face_utils.c
                        (crop+align)  (MobileFaceNet) (cosine sim)
                              │             │              │
                              └─────────────┴──────────────┘
                                            │
                                    target_embedding.c
                                    (banque + moyenne)
```

---

## Glossaire

| Terme | Définition |
|---|---|
| **NPU** | Neural Processing Unit — accélérateur matériel pour réseaux de neurones |
| **ATON** | Artificial Tensor Operations on NPU — le runtime ST pour le NPU |
| **DCMIPP** | Digital Camera Interface & Pixel Pipeline — ISP matériel du STM32N6 |
| **FSBL** | First Stage Boot Loader — premier code exécuté au démarrage |
| **SSFI** | Secure Software Firmware Image — format de signature du firmware |
| **XSPI** | eXtended SPI — bus haute vitesse pour flash/RAM externe |
| **PSRAM** | Pseudo Static RAM — mémoire externe rapide (16 MB) |
| **CHW** | Channel-Height-Width — format mémoire des tenseurs (canal en premier) |
| **HWC** | Height-Width-Channel — format mémoire des images (pixel en premier) |
| **NMS** | Non-Maximum Suppression — algorithme pour éliminer les détections redondantes |
| **IoU** | Intersection over Union — mesure de chevauchement entre deux boîtes |
| **Embedding** | Vecteur numérique représentant l'identité d'un visage |
| **Similarité cosinus** | Mesure de proximité entre deux vecteurs (angle entre eux) |
| **Epoch block** | Unité d'exécution du NPU (un sous-ensemble du graphe du réseau) |
| **Memory-mapped** | Accès à la flash/RAM externe comme de la mémoire normale via des adresses |
| **Lazy loading** | Chargement différé — le réseau de reconnaissance n'est chargé que quand nécessaire |


