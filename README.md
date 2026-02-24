# TODO
- finir le readme
- parler de l'idée de faire une suité d'image puis que le SAV forme une histoire style bd de ces images (avec bulles de texte)
- estimation du temps que ça doit prendre
- explication des choix (IA, programmes)

#  Photo Booth IA - Mode Histoire (Mini BD en 3 cases)

PhotoBooth interactif contrôlé **uniquement par gestes** qui capture **3 photos successives**, applique une transformation IA cohérente pour obtenir un rendu “bande dessinée”, puis génère une **planche BD finale** avec **bulles de texte**.  
Le but est de transformer une séance photobooth classique en une **mini-histoire en 3 scènes** (début → action → fin), rendue comme une vraie BD.  

## Objectifs du projet

- Créer une **expérience narrative** simple, fun, et répétable.
- Proposer un rendu final “collector” (planche BD) plutôt qu’une image unique.
- Assurer la cohérence stylistique des 3 panels.
- Placer des bulles de texte **sans masquer les personnes**.

---

## Fonctionnalités principales

### 1) Interaction 100% gestuelle (sans overlay)
Le système n’utilise **aucun overlay** (pas de cadre, pas de zones, pas d’aide visuelle).
L’interface est minimale : affichage plein écran du flux caméra, puis des captures, puis du résultat final.

Gestes utilisés (maintien ~2 secondes) :
- **Signe V** : déclencher la capture de la photo courante
- **Pouce vers le haut** : valider la photo et passer à la suivante
- **Pouce vers le bas** : rejeter la photo et la reprendre

### 2) Séquence “Histoire” en 3 photos
Le photobooth capture et valide 3 photos successives :
- Photo 1 → Panel 1 (début)
- Photo 2 → Panel 2 (action)
- Photo 3 → Panel 3 (fin)

L’utilisateur peut refaire une photo tant qu’elle n’est pas validée.

### 3) Génération IA cohérente (3 panels)
Une fois les 3 photos validées, l’IA applique un style BD **cohérent entre les 3 images** :
- mêmes paramètres de génération
- cohérence stylistique (palette, traits, ambiance)
- cohérence globale (même “univers” sur les 3 panels)

### 4) Bulles de texte (histoires fixes) + placement adaptatif
La planche finale contient des **bulles de texte** pour renforcer l’effet BD.

- Le texte provient de **4 à 5 histoires fixes** (templates) stockées dans un fichier de configuration.
- Le **placement des bulles s’adapte automatiquement** à la position des personnes sur chaque photo :
  - on détecte les personnes (MediaPipe Pose, optionnel Face)
  - on évite de recouvrir visages et corps
  - on choisit une zone “libre” parmi des positions candidates (coins / centres)

### 5) Composition automatique de la planche BD
Le système assemble :
- les 3 panels stylisés
- les bulles
- un **titre** (lié à l’histoire choisie)
dans une seule image finale prête à afficher et à imprimer/exporter.

---

## Parcours utilisateur (expérience)

1. **Écran caméra** (plein écran)
2. **Signe V** (2s) → Capture Photo 1
3. **Écran preview Photo 1**
   - **Pouce vers le haut** (2s) → valider et passer à Photo 2
   - **Pouce vers le bas** (2s) → refaire Photo 1
4. Idem pour Photo 2
5. Idem pour Photo 3
6. **Traitement IA** (génération des 3 panels)
7. Ajout des **bulles adaptatives**
8. Assemblage et affichage de la **planche BD finale**
9. **Pouce vers le haut** (2s) → impression puis nouvelle session  
   **Pouce vers le bas** (2s) → recommencer la séquence complète

---

## Fichiers générés (sorties)

Le photobooth produit au minimum :

- `outputs/session_YYYYMMDD_HHMMSS/`
  - `inputs/`
    - `input_1.png`
    - `input_2.png`
    - `input_3.png`
  - `ai/`
    - `panel_1.png`
    - `panel_2.png`
    - `panel_3.png`
  - `bubbles/`
    - `panel_1_bubbled.png`
    - `panel_2_bubbled.png`
    - `panel_3_bubbled.png`
  - `final/`
    - `comic_final.png`
    - *(optionnel)* `comic_final.pdf`
  - `metadata.json` (histoire choisie, seed, paramètres, timestamps)

---


# Architecture complète du système

Cette section décrit **l’architecture complète** du PhotoBooth IA “Mode Histoire” (3 cases BD).

---

## Couches du système

### Couche matérielle
| Composant | Recommandation |
|---|---|
| **GPU** | NVIDIA CUDA (RTX 2060+ recommandé pour SDXL) |
| **Webcam** | 720p minimum (1080p recommandé), 30 FPS |
| **CPU** | 6 cœurs minimum (détection gestes + I/O) |
| **RAM** | 16 GB min, 32 GB recommandé |
| **Écran** | 1920×1080 min |
| **Imprimante** | compatible CUPS / pilote système |

### Système d’exploitation
- Linux (Ubuntu/Debian recommandé) ou Windows
- Drivers webcam (V4L2 sous Linux)
- Drivers GPU NVIDIA + CUDA
- CUPS pour l’impression sous Linux

### Environnement logiciel
- Python 3.10 (recommandé pour compatibilité stable côté SD WebUI)
- Un environnement virtuel dédié (venv/conda)
- Stable Diffusion WebUI (Automatic1111) lancé en **process séparé**

---

## Vue d’ensemble (diagramme)

```text
================================================================================
                          PHOTOBOOTH IA — MODE HISTOIRE
================================================================================

COUCHE MATERIELLE
-----------------
    Webcam USB                 GPU NVIDIA CUDA                       Imprimante
  1280x720@30fps                SDXL VRAM 9-12GB                      A6/A5/A4
        |                              |                                |
        +------------------------------+--------------------------------+
                                       |
SYSTEME D'EXPLOITATION
----------------------
Linux/Windows
- Drivers webcam
- Drivers NVIDIA
- CUPS

ENV PYTHON 3.10 (venv)
----------------------
OpenCV + MediaPipe + NumPy + Requests + Pillow
            |
            v
================================================================================
                          APPLICATION PRINCIPALE (photobooth_story.py)
================================================================================
MODULE 1: CAPTURE CAMERA
    OpenCV VideoCapture() -> frames -> affichage plein écran

MODULE 2: DETECTION GESTES
    MediaPipe Hands -> classification geste (V / ThumbUp / ThumbDown)
    + logique "maintien 2 secondes"

MODULE 3: MACHINE A ETATS (3 PHOTOS)
    IDLE -> CAPTURE_1 -> PREVIEW_1 -> CAPTURE_2 -> PREVIEW_2 -> CAPTURE_3 -> PREVIEW_3
    puis PROCESSING -> DISPLAY_RESULT

MODULE 4: GENERATION IA (SDXL img2img)
    HTTP POST /sdapi/v1/img2img (Automatic1111)
    3 requêtes (1 par panel) avec paramètres cohérents (seed, prompt, CN pose)

MODULE 5: DETECTION PERSONNES (pour bulles)
    MediaPipe Pose (et option Face) sur panel_X.png
    -> zones "à éviter" (bounding boxes)

MODULE 6: RENDU BULLES
    Pillow : dessine bulles + texte (stories.json) placement adaptatif

MODULE 7: COMPOSITION PLANCHE BD
    Pillow : assemble 3 panels bubbled -> comic_final.png (+ pdf option)

================================================================================
                          STABLE DIFFUSION WEBUI (Automatic1111)
================================================================================
Process séparé:
- Port local 127.0.0.1:7860
- API REST activée
- SDXL + ControlNet OpenPose
```

---

## Machine à états (interaction gestuelle)

L’application repose sur une machine à états claire, indispensable pour :  
- guider la capture des 3 photos  
- permettre “valider” ou “refaire”  
- déclencher la génération IA uniquement quand tout est validé  
- simplifier le debug et les logs  

États:  
```text
[IDLE]
  |
  | (V ✌️ maintenu 2s)
  v
[CAPTURE_1] -> [PREVIEW_1]
                 |--(👍 2s)--> [CAPTURE_2] -> [PREVIEW_2]
                 |--(👎 2s)--> [CAPTURE_1]

[PREVIEW_2]
  |--(👍 2s)--> [CAPTURE_3] -> [PREVIEW_3]
  |--(👎 2s)--> [CAPTURE_2]

[PREVIEW_3]
  |--(👍 2s)--> [PROCESSING] -> [DISPLAY_RESULT]
  |--(👎 2s)--> [CAPTURE_3]

[DISPLAY_RESULT]
  |--(👍 2s)--> retour [IDLE] (nouvelle session)
  |--(👎 2s)--> reset complet -> [CAPTURE_1]

```
---

Principes d’implémentation

Boucle principale à ~30 FPS (dépend webcam)

À chaque frame :

lire frame caméra

détecter geste

mettre à jour un timer de “maintien”

valider geste si tenu ≥ 2s

appliquer transition d’état

Les états CAPTURE_X ne durent qu’une frame :

capture instantanée

écriture sur disque

passage immédiat à PREVIEW_X

✋ Détection de gestes (sans overlay)
Dépendances

mediapipe (Hands)

opencv-python (capture caméra)

numpy

Gestes attendus

✌️ Victory (index + majeur levés)

👍 Thumb_Up

👎 Thumb_Down

Validation par maintien (~2 secondes)

La détection brute varie frame-to-frame. On impose donc une règle :

Un geste est “validé” si :

il est détecté consécutivement pendant HOLD_TIME_SEC (ex: 2.0s)

avec une tolérance d’erreur faible (ex: 2 frames max manquées)

Pseudo-logique :

si geste courant == geste précédent : incrémenter compteur

sinon : reset compteur

valider quand compteur >= HOLD_TIME_SEC * FPS_ESTIME

Paramètres recommandés :

HOLD_TIME_SEC = 2.0

FPS_ESTIME = 30

MAX_MISSED_FRAMES = 2

📸 Capture & affichage
Capture

OpenCV VideoCapture(0)

résolution recommandée : 1280×720

format BGR (OpenCV) converti en RGB uniquement si nécessaire (Pillow / MediaPipe)

Affichage minimaliste

fenêtre plein écran “Camera” pendant IDLE / capture

fenêtre plein écran “Preview” pendant PREVIEW_X

fenêtre plein écran “Result” pendant DISPLAY_RESULT

⚠️ Aucun overlay (pas de barres de progression ni d’icônes).
Le seul “feedback” est le changement de mode d’affichage (caméra vs preview vs résultat).

🧠 Pipeline IA (vue globale)

Le module IA doit transformer chaque input_X.png en panel_X.png avec un style cohérent.

Principe

3 requêtes img2img (une par panel)

paramètres identiques (prompt / negative prompt / sampler / steps / cfg)

seed gérée pour cohérence (voir étape 3)

ControlNet OpenPose pour conserver posture/structure

Découplage recommandé

L’application photobooth ne charge pas SDXL

Elle appelle l’API d’Automatic1111 via HTTP :

plus stable

redémarrable

logs séparés

💬 Placement adaptatif des bulles (vue architecture)

Après génération IA des panels, on ajoute des bulles en fonction des personnes présentes.

Entrées

panel_X.png (image IA)

histoire choisie : stories.json (texte fixe)

style bulles : bubble_style.yaml

Étapes

Détecter la/les personnes sur panel_X.png :

MediaPipe Pose (option Face)

Convertir landmarks -> bounding boxes “zones à éviter”

Générer des rectangles de bulles (selon texte + police)

Tester plusieurs positions candidates

Choisir la meilleure (score minimum)

Dessiner bulle + contour + queue + texte via Pillow

Export : panel_X_bubbled.png

🧷 Composition de la planche BD

Le module “Composer” assemble les 3 panels “bubbled” dans une planche finale.

Entrées

panel_1_bubbled.png

panel_2_bubbled.png

panel_3_bubbled.png

titre (depuis l’histoire choisie)

Sorties

comic_final.png

optionnel : comic_final.pdf

Layout recommandé (simple)

3 panels alignés horizontalement (ou vertical selon imprimante)

marges externes + gouttière entre cases

titre en haut

Tous les paramètres (taille, marges) doivent être configurables.





## Logs & traçabilité (recommandé)

Chaque session doit écrire un metadata.json contenant :

timestamp session

histoire choisie (id + titre)

chemins des images input/panel/final

seed utilisée

paramètres SDXL (steps, cfg, denoise, sampler, model)

paramètres bulles (font, size, positions retenues)

C’est indispensable pour :

reproduire un bug

comparer des réglages IA

auditer la cohérence des sorties

🔌 Process séparés (recommandation forte)

Le projet tourne idéalement avec 2 terminaux/process :

Terminal 1 — Stable Diffusion WebUI (Automatic1111)

lance le serveur API sur 127.0.0.1:7860

charge SDXL + ControlNet

Terminal 2 — PhotoBooth Story

lance python photobooth_story.py

capture webcam + contrôle gestes + appels API

Cette séparation facilite :

la stabilité

la relance en cas de crash

le debug


























# Résultat attendu
### Les photo prisent par le SAV
<img width="400" height="400" alt="image1" src="https://github.com/user-attachments/assets/3cf57bd1-2696-4c26-afb2-bfad7aee3bad" />
<img width="400" height="400" alt="image2" src="https://github.com/user-attachments/assets/bb63f815-e16f-46b4-a4d2-ef36b89ff7a3" />
<img width="183" height="275" alt="image3" src="https://github.com/user-attachments/assets/a0178629-9200-4716-adc4-1eac74ea53e0" />

### Exemple de ce que le SAV doit renvoyer
<img width="400" height="400" alt="image_final" src="https://github.com/user-attachments/assets/2e951b0a-e5db-4479-b126-0dfc58e03814" />

