# TECHNICAL DESIGN DOCUMENT (TDD) : SpinTheTrack

**Version :** 2.1 (Multi-Repo Architecture)
**Date :** 2026-02-13
**Author :** AI Architect
**Project Status :** Initialization Phase

---

## 1. EXECUTIVE SUMMARY

**SpinTheTrack** est une application hybride (Phygital) de quiz musical.
Le système repose sur une architecture **micro-services conteneurisée**, conçue pour être déployée localement ou sur un serveur cloud léger.

L'objectif de ce document est de définir les spécifications techniques pour le développement du MVP (Minimum Viable Product). La priorité est donnée à la modularité via une stratégie **Multi-Repo**, une séparation stricte Frontend/Backend, et une gestion des données sans base de données persistante (NoSQL/SQL) pour l'instant.

---

## 2. REPOSITORY STRATEGY & GITHUB ORGANIZATION

Pour garantir une isolation totale du code et faciliter la CI/CD future, le projet sera hébergé sous une **Organisation GitHub** dédiée (ex: `github.com/SpinTheTrack-Studio`) contenant 3 dépôts distincts.

### 2.1 Structure des Dépôts

1.  **`spinthetrack-backend`** (Service)
    * Contient uniquement la logique métier Python, l'API FastAPI et les données statiques (JSON).
    * Cycle de vie indépendant.
2.  **`spinthetrack-frontend`** (Service)
    * Contient uniquement l'application React/Vite.
    * Cycle de vie indépendant.
3.  **`spinthetrack-app`** (Orchestrateur / Infrastructure)
    * Agit comme le "Parent".
    * Utilise **Git Submodules** pour référencer des commits précis du Back et du Front.
    * Contient la configuration Docker Compose globale pour lancer le projet.

---

## 3. ARCHITECTURE SYSTÈME

### 3.1 Diagramme Logique (Docker)

L'application tourne via `docker-compose` orchestrant deux services principaux communiquant via un réseau privé Docker.

```text
[ CLIENT / NAVIGATEUR ]
       |
       |  Requêtes HTTP (Port 3000 pour UI, via Proxy interne vers API)
       v
+-------------------------------------------------------+
|  DOCKER HOST (Votre Machine / Serveur)                |
|                                                       |
|  +--------------------------+    +------------------+ |
|  | SERVICE: FRONTEND        |    | SERVICE: BACKEND | |
|  | (Image: Node/Nginx)      |    | (Image: Python)  | |
|  |                          |    |                  | |
|  | - React App (Vite)       |    | - FastAPI App    | |
|  | - QR Scanner Library     |<-->| - Game Logic     | |
|  | - Animations (Framer)    |    | - Data (JSON)    | |
|  | - Deezer Web SDK         |    | - Memory Store   | |
|  +--------------------------+    +------------------+ |
|            ^                              ^           |
+------------|------------------------------|-----------+
             |                              |
      (HTTPS Calls)                  (Read-Only)
             v                              v
 [ API DEEZER PUBLIQUE ]           [ cards.json ]
```

### 3.2 Stack Technologique

| Composant | Technologie | Justification |
| :--- | :--- | :--- |
| **Backend** | **Python (FastAPI)** | Haute performance (Asynchrone), Validation stricte des données (Pydantic), Documentation automatique (Swagger UI). |
| **Frontend** | **React (Vite)** | Écosystème riche, Build ultra-rapide, Gestion d'état complexe pour les animations de jeu. |
| **Data** | **JSON + RAM** | Pas de SGBD (PostgreSQL/Mongo) pour le MVP. Les cartes sont statiques (JSON), l'état de la partie est volatile (RAM). |
| **Container** | **Docker Compose** | Orchestration simple pour le développement et le déploiement. |
| **Protocol** | **REST (HTTP)** | Communication simple et robuste. WebSockets jugés inutiles pour ce type d'interaction "Request-Response". |

---

## 4. GESTION DES DONNÉES (NO-DB STRATEGY)

Conformément aux contraintes, aucune base de données persistante n'est utilisée.

### 4.1 Données Statiques (Reference)
Le fichier `cards.json` (situé dans le repo Backend) est la "Source de Vérité". Il est chargé en RAM au démarrage du conteneur Backend.

```json
// Exemple de structure cards.json
[
  {
    "id": "STT_001",
    "deezer_id": "145632",
    "meta": { "artist": "Daft Punk", "title": "One More Time" },
    "weights": { "chrono": 0.2, "blind": 0.5, "lyrics": 0.3 }
  }
]
```

### 4.2 Données Volatiles (State)
L'état de la partie est stocké dans une variable globale Python (Dictionnaire) au sein du service Backend.
* *Avantage :* Extrême rapidité (zéro latence I/O).
* *Inconvénient :* Si le conteneur Backend redémarre, la partie en cours est perdue (Acceptable pour le MVP).

---

## 5. API SPECIFICATIONS (DRAFT)

Le Frontend communiquera avec le Backend via ces endpoints principaux :

* **`GET /health`**
    * Vérifie que le backend est en ligne et que le fichier JSON est chargé.
* **`POST /api/game/scan`**
    * **Input :** `{ "qr_code": "STRING_ID" }`
    * **Process :** Valide le code, calcule le mode de jeu (RNG pondéré), récupère les métadonnées.
    * **Output :** Objet JSON complet pour le round (Mode, Track Info, Challenge Data).
* **`POST /api/game/validate`** (Optionnel pour V1)
    * Permet de valider un point côté serveur pour garder le score en mémoire.

---

## 6. STRUCTURE DES DOSSIERS (FILE TREE)

Voici l'organisation physique sur votre machine de développement.

### Repo 1: `spinthetrack-app` (Root)
```text
spinthetrack-app/
├── .gitmodules             # Configuration des sous-modules
├── docker-compose.yml      # Orchestration globale
└── README.md               # Documentation de lancement
```

### Repo 2: `spinthetrack-backend` (Submodule)
```text
backend/
├── Dockerfile              # Image Python Slim
├── .dockerignore
├── requirements.txt        # fastapi, uvicorn, pydantic, requests
└── app/
    ├── __init__.py
    ├── main.py             # Entry point FastAPI
    ├── config.py           # Variables d'env
    ├── core/
    │   ├── logic.py        # RNG & Game Mechanics
    │   └── models.py       # Pydantic Schemas (Card, Response)
    └── data/
        └── cards.json      # Fichier source des questions
```

### Repo 3: `spinthetrack-frontend` (Submodule)
```text
frontend/
├── Dockerfile              # Multi-stage build (Node -> Nginx)
├── .dockerignore
├── nginx.conf              # Conf Nginx pour le container
├── package.json
├── vite.config.js
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── components/
    │   ├── shared/         # Boutons, Layouts
    │   ├── scanner/        # QR Reader Logic
    │   └── modes/          # Composants (LyricsView, BlindView...)
    └── services/
        └── api.js          # Axios config
```

---

## 7. PLAN D'IMPLÉMENTATION (ROADMAP)

### Phase 1 : Infrastructure (Jour 1)
1.  Créer l'Organisation GitHub.
2.  Créer les 3 repos vides.
3.  Initialiser `spinthetrack-app` et lier les sous-modules (`git submodule add...`).
4.  Écrire le `docker-compose.yml` et les `Dockerfile` vides.
5.  **Milestone :** `docker-compose up` lance deux conteneurs qui ne crashent pas.

### Phase 2 : Backend Logic (Jour 2)
1.  Peupler `cards.json` avec 5 cartes de test.
2.  Créer les modèles Pydantic (`Card`, `GameRound`).
3.  Implémenter l'endpoint `/scan` avec la logique RNG.
4.  **Milestone :** Tester l'API via Swagger UI (`http://localhost:8000/docs`) -> Le scan renvoie bien un mode de jeu aléatoire.

### Phase 3 : Frontend Base (Jour 3)
1.  Scaffolding React + Vite.
2.  Implémenter le routeur basique.
3.  Intégrer le lecteur QR Code.
4.  Connecter le scanner à l'API Backend.
5.  **Milestone :** Je scanne un code -> La console du navigateur affiche l'objet JSON reçu du Back.

### Phase 4 : Frontend UX/UI (Jour 4-5)
1.  Créer l'animation "Spin" (Roue de la fortune) avec Framer Motion.
2.  Développer les 3 vues de jeu (Chrono / Blind / Lyrics).
3.  Intégrer le SDK Deezer pour lancer la musique *après* l'animation.

### Phase 5 : Polishing (Jour 6)
1.  Gestion des erreurs (Code inconnu, Pas de réseau).
2.  Tests d'intégration complets.