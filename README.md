# Traducteur et Reconnaissance Vocale (Éwé & Gegbe)

Ce projet vise à développer une solution complète de reconnaissance vocale et de traduction pour les langues **Éwé** et **Gegbe (Mina)** (langues vernaculaires parlées principalement au Togo, au Ghana et au Bénin). L'objectif final est de permettre la conversion de la parole en texte (Speech-to-Text) et la traduction bidirectionnelle entre ces langues et le Français ou l'Anglais, le tout via une interface graphique intuitive.



## ✨ Fonctionnalités Clés Implémentées

Le projet intègre nativement les trois piliers suivants, fonctionnels et testés :

### 1. 🎙️ Reconnaissance Vocale (ASR Unifié)
-   **Objectif** : Convertir la parole (Mina ou Éwé) en texte brut.
-   **Technologie** : Modèle **OpenAI Whisper** (architecture Transformer).
-   **Implémentation** : 
    -   Nous utilisons une approche unifiée où le modèle est capable de transcrire les deux langues.
    -   Entraînement optimisé pour CPU via le script `src/models/train_whisper_cpu.py`.
    -   **Données** : Le modèle est entraîné sur **notre propre corpus biblique** constitué localement par nos scrapers. Il combine :
        -   Audio/Texte **Éwé** (Bible complète).
        -   Audio/Texte **Gegbe** (Bible complète).
        -   Ces données sont fusionnées dans `data/processed/bible_asr_dataset.csv`.

### 2. 🔄 Chaîne de Traduction en Cascade (Pivot Strategy)
Une architecture sophistiquée en deux temps pour pallier le manque de données directes Mina-Français :
-   **Étape A : Mina ➔ Éwé (Normalisation)**
    -   **Modèle** : `facebook/nllb-200` (No Language Left Behind).
    -   **Rôle** : Utilise l'Éwé comme langue pivot standardisée. Le modèle est capable de comprendre le Mina (proche dialectalement) et de le reformuler en Éwé écrit standard.
-   **Étape B : Éwé ➔ Français (Traduction Finale)**
    -   **Modèle** : `Helsinki-NLP/opus-mt-ee-fr`.
    -   **Rôle** : Modèle de traduction neuronale spécialisé (NMT) qui assure une haute qualité linguistique vers le français.

### 3. ⚡ Optimisation CPU & Inférence
Le projet est conçu pour tourner sur des machines sans GPU (ex: laptops étudiants) :
-   **CTranslate2** : Moteur d'inférence ultra-rapide intégré pour le modèle NMT (`src/models/translation_ewe_fr.py`). Il permet une exécution 2x à 4x plus rapide qu'un modèle PyTorch standard sur CPU.
-   **Quantification INT8** : Réduction de la précision des poids (de 32 bits à 8 bits) pour diviser par 4 la consommation mémoire sans perte notable de qualité.
-   **Ready-to-use** : L'infrastructure supporte l'ajout futur de `faster-whisper` pour la partie vocale.

## Vision du Projet

Le système final offrira les fonctionnalités suivantes :
1.  **Reconnaissance Vocale (ASR) :** Conversion de fichiers audio Mina/Éwé en texte via un modèle Whisper unifié.
2.  **Chaîne de Traduction en Cascade :**
    -   Mina ➔ Éwé (Adaptation dialectale via NLLB fine-tuné).
    -   Éwé ➔ Français / Autre (via Helsinki-NLP Opus-MT).
3.  **Optimisation CPU :** Utilisation de la quantification **INT8**, **faster-whisper** et **CTranslate2** pour garantir des performances fluides sur du matériel grand public (CPU).
4.  **Interface Graphique (GUI) :** Application interactive pour la saisie audio/texte et l'affichage des traductions.

## Architecture & Logique du Modèle

Ce système repose sur une **architecture en cascade** (Cascaded Architecture) composée de trois intelligences artificielles distinctes qui communiquent entre elles pour réaliser la tâche finale.

### Le Flux de Données
1.  **Entrée** : Audio (Parole en Mina ou Éwé).
2.  **Transcription (ASR)** : Le son est converti en texte Mina/Éwé.
3.  **Pivot (Si nécessaire)** : Si la langue source est le Mina, le texte est traduit vers l'Éwé.
4.  **Traduction Finale (NMT)** : Le texte Éwé est traduit en Français.

### Les Modèles Composants

#### 1. ASR - Reconnaissance Vocale (`openai/whisper`)
-   **Rôle** : Convertir l'audio en texte.
-   **Modèle** : Nous utilisons **OpenAI Whisper (Base/Small)**, fine-tuné sur notre corpus biblique local.
-   **Lien avec le code** : Géré par `src/models/train_whisper_cpu.py`.
-   **Importance** : C'est la "bouche" du système. Sans lui, impossible de traiter la parole. Nous l'avons optimisé pour le CPU en gelant 90% de ses paramètres durant l’entraînement.

#### 2. Pivot - Adaptation Dialectale (`facebook/nllb-200`)
-   **Rôle** : Combler le fossé entre le Mina et l'Éwé.
-   **Modèle** : **NLLB-200 (No Language Left Behind)** de Meta.
-   **Logique** : Le Mina manque de ressources directes vers le français. L'Éwé étant une langue sœur très proche avec plus de ressources, nous utilisons NLLB pour "normaliser" le Mina en Éwé standard.
-   **Lien avec le code** : Implémenté dans `src/models/translation_mina_ewe.py`.

#### 3. NMT - Traduction Finale (`Helsinki-NLP/opus-mt-ee-fr`)
-   **Rôle** : Traduire le texte Éwé vers le Français.
-   **Modèle** : **OPUS-MT** de l'Université d'Helsinki.
-   **Importance** : C'est le modèle spécialisé qui connaît la grammaire française. Il assure la fluidité de la sortie finale.
-   **Lien avec le code** : Utilisé dans `src/models/translation_ewe_french.py`.

### Orchestration (`TranslationCascade`)
Le fichier `src/pipeline/translate_cascade.py` est le chef d'orchestre. Il initialise les trois modèles et fait passer les données de l'un à l'autre de manière transparente pour l'utilisateur.

### ❓ Pourquoi cette architecture complexe ?

Pourquoi ne pas faire simplement **Audio Mina ➔ Texte Français** ?

1.  **Manque de Données (Low-Resource)** : Il n'existe pas de dataset massif de type "Audio Mina ↔ Texte Français" pour entraîner une IA directe.
2.  **La Stratégie Pivot** : Nous disposons de la Bible en Mina et en Éwé. Comme ces langues sont très proches, nous pouvons traduire le Mina en Éwé (plus riche en ressources).
3.  **Le Rôle de l'ASR** : L'ASR est **indispensable**. C'est le seul moyen de passer du monde sonore au monde textuel. Sans lui, les modèles de traduction (qui ne lisent que du texte) seraient inutilisables pour une application vocale.

## Structure du Projet (Code)

```text
├── data/
│   ├── raw/                # Données brutes de scraping
│   ├── processed/          # Corpus parallèle aligné (Mina/Éwé)
├── models/                 # Stockage local des modèles (Whisper, NLLB, OPUS)
├── notebooks/              # Travaux d'expérimentation
├── src/
│   ├── config/             # Hyperparamètres (batch size, freeze, etc.)
│   ├── models/             # C'est ici que vivent les modèles :
│   │   ├── train_whisper_cpu.py    # Entraînement ASR
│   │   ├── translation_mina_ewe.py # Logique du Pivot (NLLB)
│   │   └── translation_ewe_french.py # Logique NMT Finale
│   ├── pipeline/           # Orchestration (translate_cascade.py)
│   ├── preprocessing/      # Alignement et nettoyage de corpus
│   └── scraping/           # Scripts de collecte Bible
└── requirements.txt        # Dépendances Python
```

## État d'avancement technique

## Installation

### 1. Prérequis
- Python 3.8+
- Bibliothèques additionnelles (voir `requirements.txt`)

### 2. Mise en place
```bash
# Créer et activer l'environnement virtuel
python -m venv venv
# Windows
venv\Scripts\activate
# Linux/Mac
source venv/bin/activate

# Installer les dépendances (IMPORTANT)
pip install -r requirements.txt
```

## Guide d'Utilisation (Pas-à-pas)

Pour mener à bien le projet de la collecte à la traduction finale, suivez ces étapes dans l'ordre :

### Étape 1 : Collecte des données
Lancez le scraper pour récupérer les textes et audios des bibles Éwé et Mina.
```bash
python -m src.pipeline.build_corpus
```
*Cette étape crée aussi automatiquement le fichier d'alignement `data/processed/parallel_mina_ewe.csv`.*

### Étape 2 : Préparation du Dataset ASR
Ouvrez et exécutez le notebook **`notebooks/02_prepare_asr_dataset.ipynb`**. 
- Il convertira les audios en WAV 16kHz.
- Il nettoiera les textes.
- Il générera le dataset final pour l'entraînement.

### Étape 3 : Entraînement Local (CPU)
Ouvrez le notebook **`notebooks/03_train_whisper_ewe.ipynb`**.
- Il appelle le module `src.models.train_whisper_cpu`.
- Vous pouvez y modifier les hyperparamètres (gel des couches, batch size) et suivre l'avancement de l'apprentissage ASR.

### Étape 4 : Traduction Finale (Cascade)
Utilisez le même notebook ou le terminal pour tester la chaîne complète :
```bash
python -m src.pipeline.translate_cascade
```
*Le système prendra une phrase en Mina, la pivotera en Éwé, puis la traduira en Français.*

## Configuration du Matériel
Le projet est optimisé pour tourner sur **CPU uniquement**. 
- Inférence : **INT8** via CTranslate2/faster-whisper.
- Entraînement : **Paramètres gelés à 90%** pour économiser la RAM et le processeur.



---
*Ce projet est réalisé dans un cadre académique et de recherche pour la promotion des langues vernaculaires à travers les technologies de l'IA.*
