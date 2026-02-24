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
  | (signe V  maintenu 2s)
  v
[CAPTURE_1] -> [PREVIEW_1]
                 |--(pouce levé 2s)--> [CAPTURE_2] -> [PREVIEW_2]
                 |--(pouce baissé 2s)--> [CAPTURE_1]

[PREVIEW_2]
  |--(pouce levé 2s)--> [CAPTURE_3] -> [PREVIEW_3]
  |--(pouce baissé 2s)--> [CAPTURE_2]

[PREVIEW_3]
  |--(pouce levé 2s)--> [PROCESSING] -> [DISPLAY_RESULT]
  |--(pouce baissé 2s)--> [CAPTURE_3]

[DISPLAY_RESULT]
  |--(pouce levé 2s)--> retour [IDLE] (nouvelle session)
  |--(pouce baissé 2s)--> reset complet -> [CAPTURE_1]

```

---

## Principes d’implémentation

Boucle principale à ~30 FPS (dépend webcam)  

À chaque frame :  
- lire frame caméra  
- détecter geste  
- mettre à jour un timer de “maintien”  
- valider geste si tenu ≥ 2s  
- appliquer transition d’état  

Les états CAPTURE_X ne durent qu’une frame :  
- capture instantanée  
- écriture sur disque  
- passage immédiat à PREVIEW_X  

### Détection de gestes 
Dépendances:  
- mediapipe (Hands)  
- opencv-python (capture caméra)  
- numpy  

Validation par maintien (~2 secondes)  

La détection brute varie frame-to-frame. On impose donc une règle :  

Un geste est “validé” si :  
- il est détecté consécutivement pendant HOLD_TIME_SEC (ex: 2.0s)  
- avec une tolérance d’erreur faible (ex: 2 frames max manquées)  

Pseudo-logique :  
- si geste courant == geste précédent : incrémenter compteur  
- sinon : reset compteur  
- valider quand compteur >= HOLD_TIME_SEC * FPS_ESTIME  

Paramètres recommandés :
- HOLD_TIME_SEC = 2.0  
- FPS_ESTIME = 30  
- MAX_MISSED_FRAMES = 2  

---

## Capture & affichage
### Capture
- OpenCV VideoCapture(0)
- résolution recommandée : 1280×720
- format BGR (OpenCV) converti en RGB uniquement si nécessaire (Pillow / MediaPipe)

### Affichage minimaliste  
- fenêtre plein écran “Camera” pendant IDLE / capture  
- fenêtre plein écran “Preview” pendant PREVIEW_X  
- fenêtre plein écran “Result” pendant DISPLAY_RESULT  
- Le seul “feedback” est le changement de mode d’affichage (caméra vs preview vs résultat).  

---

## Pipeline IA (vue globale)

Le module IA doit transformer chaque input_X.png en panel_X.png avec un style cohérent.

### Principe

- 3 requêtes img2img (une par panel)  
- paramètres identiques (prompt / negative prompt / sampler / steps / cfg)  
- seed gérée pour cohérence (voir étape 3)  
- ControlNet OpenPose pour conserver posture/structure  

### Découplage recommandé

- L’application photobooth ne charge pas SDXL  

Elle appelle l’API d’Automatic1111 via HTTP :  
- plus stable  
- redémarrable  
- logs séparés  

---

## Placement adaptatif des bulles (vue architecture)

Après génération IA des panels, on ajoute des bulles en fonction des personnes présentes.

### Entrées

- panel_X.png (image IA)  
- histoire choisie : stories.json (texte fixe)  
- style bulles : bubble_style.yaml  

### Étapes

1°) Détecter la/les personnes sur panel_X.png :  
- MediaPipe Pose (option Face)  

2°) Convertir landmarks -> bounding boxes “zones à éviter”

3°) Générer des rectangles de bulles (selon texte + police)

4°) Tester plusieurs positions candidates

5°) Choisir la meilleure (score minimum)

6°) Dessiner bulle + contour + queue + texte via Pillow

7°) Export : panel_X_bubbled.png

---

## Composition de la planche BD

Le module “Composer” assemble les 3 panels “bubbled” dans une planche finale.

### Entrées

- panel_1_bubbled.png  
- panel_2_bubbled.png  
- panel_3_bubbled.png  
- titre (depuis l’histoire choisie)  

### Sorties

- comic_final.png  
- optionnel : comic_final.pdf  

### Layout recommandé (simple)

- 3 panels alignés horizontalement (ou vertical selon imprimante)  
- marges externes + gouttière entre cases  
- titre en haut  
- Tous les paramètres (taille, marges) doivent être configurables.  

---

## Logs & traçabilité

Chaque session doit écrire un metadata.json contenant :
- timestamp session  
- histoire choisie (id + titre)  
- chemins des images input/panel/final  
- seed utilisée  
- paramètres SDXL (steps, cfg, denoise, sampler, model)  
- paramètres bulles (font, size, positions retenues)  
 
C’est indispensable pour :
- reproduire un bug  
- comparer des réglages IA  
- auditer la cohérence des sorties  

### Process séparés

Le projet tourne idéalement avec 2 terminaux/process :  

Terminal 1 — Stable Diffusion WebUI (Automatic1111)  
- lance le serveur API sur 127.0.0.1:7860  
- charge SDXL + ControlNet  

Terminal 2 — PhotoBooth Story  
- lance python photobooth_story.py  
- capture webcam + contrôle gestes + appels API  

Cette séparation facilite :  
- la stabilité  
- la relance en cas de crash  
- le debug  


#  — Développement & Implémentation Technique

---

#  Objectif

Cette étape correspond au cœur technique du projet.

Elle consiste à développer l’ensemble des systèmes permettant :

-  Capture intelligente de 3 photos
-  Détection du geste "V"
-  Détection du pouce levé
-  Détection du visage
-  Placement automatique des bulles
-  Application automatique du meilleur filtre
-  Génération d’une histoire cohérente (sans LLM)
-  Composition et export final

---

#  1. Architecture Générale

## 1.1 Organisation des modules

Le projet est divisé en modules indépendants pour assurer :

- Maintenabilité
- Scalabilité
- Clarté du code
- Tests unitaires simplifiés

```
/camera
    CameraManager
    CaptureManager

/gesture
    GestureDetector
    GestureStateMachine

/face
    FaceDetector
    BubblePositionManager

/ui
    OverlayManager
    CountdownManager
    TextDisplayManager

/story
    StoryEngine
    TemplateManager

/filter
    FilterManager

/export
    ImageComposer
    ExportManager
```

---

#  2. Détection des Gestes

---

## 2.1 Détection du geste en V

### Objectif :
Déclencher l’affichage du texte et le lancement du processus photo.

### Processus :

1. Analyse flux caméra en temps réel
2. Détection des landmarks de la main
3. Vérification configuration des doigts

### Logique :

```pseudo
if indexFinger == extended
AND middleFinger == extended
AND ringFinger == folded
AND pinky == folded
THEN gesture = "V"
```

---

## 2.2 Détection du pouce en l'air

### Objectif :
Valider ou rejeter la photo capturée.

```pseudo
if thumb == extended
AND otherFingers == folded
THEN gesture = "THUMBS_UP"
```

---

## 2.3 Machine d’état

Le système repose sur une machine d’état robuste :

```
WAITING_FOR_V
    ↓
TEXT_DISPLAYED
    ↓
COUNTDOWN_RUNNING
    ↓
PHOTO_CAPTURED
    ↓
WAITING_FOR_VALIDATION
    ↓
VALIDATED (photo conservée)
or
REJECTED (reprise)
```

---

#  3. Détection du Visage

---

## 3.1 Détection Bounding Box

```pseudo
faceBoundingBox = detectFace(frame)
faceCenterX = faceBoundingBox.centerX
faceCenterY = faceBoundingBox.centerY
```

---

## 3.2 Placement automatique des bulles

### Règle principale :

| Position visage | Position bulle |
|----------------|----------------|
| Haut gauche | Bas droite |
| Haut droite | Bas gauche |
| Bas gauche | Haut droite |
| Bas droite | Haut gauche |
| Centre | Zone libre dominante |

---

### Algorithme :

```pseudo
if faceCenterX < imageWidth/2 AND faceCenterY < imageHeight/2
    bubblePosition = BOTTOM_RIGHT
else if faceCenterX > imageWidth/2 AND faceCenterY < imageHeight/2
    bubblePosition = BOTTOM_LEFT
```

---

#  4. Génération de l’Histoire (Sans LLM)

---

## 4.1 Structure des templates

Le système utilise des scénarios prédéfinis :

```json
{
  "scenario_suspense": [
    "Aujourd’hui tout semblait normal...",
    "Puis quelque chose a attiré mon attention.",
    "Et c’est là que tout a changé."
  ],
  "scenario_fun": [
    "Je pensais que ce serait une journée tranquille.",
    "Mais j’avais tort.",
    "Et maintenant, je dois tout expliquer."
  ]
}
```

---

## 4.2 Sélection aléatoire

```pseudo
selectedStory = random(storyTemplates)
textPhoto1 = selectedStory[0]
textPhoto2 = selectedStory[1]
textPhoto3 = selectedStory[2]
```

---

#  5. Affichage Texte & Compte à Rebours

---

## 5.1 Séquence complète

1. Détection du signe en V
2. Affichage texte (2–3 secondes)
3. Lancement compte à rebours (5–10 secondes)
4. Capture automatique

---

## 5.2 Compte à rebours visuel

```pseudo
startCountdown(5)

display:
5
4
3
2
1
*capture*
```

---

#  6. Capture & Validation

---

## 6.1 Capture

```pseudo
captureImage()
storeTemporarily()
```

---

## 6.2 Validation utilisateur

```pseudo
if gesture == THUMBS_UP
    savePhoto()
    moveToNext()
else
    discardPhoto()
    restartCurrentPhoto()
```

---

## 6.3 Boucle jusqu’à 3 photos validées

```pseudo
while validatedPhotos < 3
    runPhotoSequence()
```

---

#  7. Application Automatique des Filtres

---

## 7.1 Analyse de luminosité

```pseudo
brightness = analyzeBrightness(image)
```

---

## 7.2 Sélection du filtre

```pseudo
if brightness < threshold
    applyFilter("BrightBoost")
else
    applyFilter("CinematicWarm")
```

---

## 7.3 Application

Tous les filtres sont appliqués AVANT la composition finale.

---

#  8. Composition Finale (SAV)

---

## 8.1 Pipeline final

Pour chaque photo :

1. Application filtre
2. Détection visage
3. Placement bulle
4. Injection texte
5. Ajout à la composition

---

## 8.2 Algorithme global

```pseudo
for each photo in validatedPhotos:
    filteredPhoto = applyFilter(photo)
    faceData = detectFace(filteredPhoto)
    bubblePosition = calculateBubblePosition(faceData)
    bubble = generateBubble(bubblePosition)
    composedPhoto = merge(filteredPhoto, bubble, text)
    finalComposition.add(composedPhoto)
```

---

## 8.3 Layout final possible

### Option 1 : Style BD vertical
- 3 images empilées

### Option 2 : Story Instagram
- 3 slides

### Option 3 : Image unique 3 panneaux

---

#  9. Export

---

## 9.1 Formats supportés

- JPG HD
- PNG
- PDF
- MP4 animé

---

## 9.2 Export HD

```pseudo
export(finalComposition, format="JPG", quality=100%)
```

---

#  10. Gestion des erreurs

---

## Cas gérés :

-  Aucun visage détecté
-  Mauvaise détection geste
-  Luminosité insuffisante
-  Timeout utilisateur
-  Multi-visage non prévu

---

# Résultat Final

À la fin de l’étape 3 :

✔ L’utilisateur déclenche avec un signe V  
✔ Texte s’affiche  
✔ Compte à rebours démarre  
✔ Photo prise  
✔ Validation avec le pouce en l'air  
✔ Répété 3 fois  
✔ Histoire illustrée générée automatiquement  

---

#  Conclusion actuelle

---

#  Perspective d'évolution








# Résultat attendu
### Les photo prisent par le SAV
<img width="183" height="275" alt="image1" src="https://github.com/user-attachments/assets/3cf57bd1-2696-4c26-afb2-bfad7aee3bad" />
<img width="183" height="275" alt="image2" src="https://github.com/user-attachments/assets/bb63f815-e16f-46b4-a4d2-ef36b89ff7a3" />
<img width="183" height="275" alt="image3" src="https://github.com/user-attachments/assets/a0178629-9200-4716-adc4-1eac74ea53e0" />

### Exemple de ce que le SAV doit renvoyer
<img width="400" height="400" alt="image_final" src="https://github.com/user-attachments/assets/2e951b0a-e5db-4479-b126-0dfc58e03814" />

