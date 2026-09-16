# TP Optimisation Docker

## Tableau Comparatif des Optimisations

| Étape | Tag de l'image | Taille (Disk Usage) | Réduction vs Baseline | Temps de Build | Description de l'optimisation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **0. Baseline** | `node-app:v0-baseline` | **1.92 GB** | - | ~61.1s | Image initiale non optimisée |
| **1. Dockerignore** | `node-app:v1-dockerignore` | **1.93 GB** | - | ~16.5s | Ajout du `.dockerignore` et filtrage du contexte |
| **2. Alpine Base** | `node-app:v2-alpine` | **222 MB** | -88.5% (~1.71 GB) | ~16.1s | Passage à `node:20-alpine` |
| **3. Layer Caching** | `node-app:v3-cache` | **222 MB** | -88.5% (~1.71 GB) | **0.7s (Cache Hit)** | Optimisation de l'ordre des layers pour exploiter le cache Docker |

---

## Détails par Étape

### Étape 0 : État Initial (Baseline)

* **Objectif** : Mesurer la taille initiale de l'application sans modification.
* **Résultat** : Taille de l'image : **1.92 GB** (Content Size : 482 MB).
* **Preuve** :

![Baseline Screenshot](docs/screenshots/01.png)

### Étape 1 : Isolation du contexte avec `.dockerignore`

* **Problème identifié** : L'envoi inutile de fichiers lourds (`node_modules` local, documentation, historique Git) saturait le contexte de build (10.69 MB) et provoquait des temps de transfert élevés. De plus, la directive `COPY node_modules` copiait des dépendances dépendantes de l'OS hôte.
* **Solution appliquée** :
  1. Création d'un fichier `.dockerignore` pour filtrer les artefacts locaux (`node_modules`, `.git`, `docs`).
  2. Suppression de la ligne `COPY node_modules ./node_modules` dans le `dockerfile` pour laisser `npm install` installer proprement les paquets Linux[cite: 2, 3].
* **Impact & Analyse** :
  * Le contexte envoyé au démon passe de **10.69 MB à 46.63 KB**.
  * Le temps de build est divisé par près de 4 (de **61.1s à 16.5s**).
  * La taille finale reste similaire (~1.93 GB) car l'image de base reste `node:latest` et les dépendances système inutiles (`build-essential`) sont toujours installées[cite: 2, 3].
* **Preuve** :

![Dockerignore Screenshot](docs/screenshots/02.png)

### Étape 2 : Migration vers une image de base légère (Alpine)

* **Problème identifié** : L'image de départ utilisait `node:latest` (basée sur une distribution Debian complète de plus de 1 GB) combinée à l'installation inutile d'outils de compilation C++ (`build-essential`) non requis en production.
* **Solution appliquée** :
  1. Remplacement de `node:latest` par `node:20-alpine` (système minimaliste de quelques mégaoctets).
  2. Suppression des commandes Debian incompatibles et superflues (`apt-get install -y build-essential...`)[cite: 3].
* **Impact & Analyse** :
  * La taille de l'image chute drastiquement de **1.93 GB à 222 MB** (gain de **~1.71 GB**).
  * La surface d'attaque en matière de sécurité est considérablement réduite grâce à la suppression de centaines de binaires et utilitaires système superflus.
* **Preuve** :

![Alpine Base Screenshot](docs/screenshots/03.png)

### Étape 3 : Optimisation du cache des couches (Layer Caching)

* **Problème identifié** : La directive `COPY . /app` était placée avant `RUN npm install`. À chaque modification d'un fichier source, la couche de copie changeait, invalidant complètement le cache Docker et forçant une réinstallation complète des dépendances.
* **Solution appliquée** :
  1. Séparation de la copie des métadonnées de dépendances (`COPY package*.json ./`).
  2. Exécution de l'installation (`RUN npm install`) avant de copier le reste du code.
  3. Copie du code source applicatif (`COPY . /app`).
* **Impact & Analyse** :
  * Lors d'une modification du code source applicatif sans changement des dépendances, la couche d'installation est récupérée instantanément du cache (`CACHED`).
  * Le temps de build chute drastiquement de **16.1s à 0.7s**.
* **Preuve** :

![Layer Caching Screenshot](docs/screenshots/04.png)