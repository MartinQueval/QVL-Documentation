# HB-Api-StatBar

<p align="center">
  <em>Bar-rating API of the QVL Hobbies domain — Node + Express 5 + MongoDB.</em><br>
  <em>API de notation de bars du domaine QVL Hobbies — Node + Express 5 + MongoDB.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

<p align="center">
  <img alt="Runtime" src="https://img.shields.io/badge/Node-%E2%89%A520.19-339933">
  <img alt="Framework" src="https://img.shields.io/badge/Express-5-000000">
  <img alt="Language" src="https://img.shields.io/badge/TypeScript-6-3178C6">
  <img alt="Database" src="https://img.shields.io/badge/MongoDB-Mongoose%209-47A248">
  <img alt="Port" src="https://img.shields.io/badge/prod-127.0.0.1%3A8311-blue">
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

Node/TypeScript service of the **Hobbies** domain backing **StatBar**, the app to rate, map and rank bars. It exposes a small REST CRUD over the `bars` resource, backed by MongoDB through Mongoose. It has **no public URL of its own**: in production it is consumed by the **HB-Front-StatBar** SPA, which relays `/api` to it same-origin.

## Stack

- **Runtime** : Node.js ≥ 20.19
- **HTTP** : Express 5 (`express@^5.2.1`) with `cors`
- **Language** : TypeScript (`typescript@^6.0.3`, CommonJS output)
- **Database** : MongoDB via Mongoose (`mongoose@^9.6.3`)
- **Config** : `dotenv` (`.env`, ignored by Git)
- **Dev/build** : `ts-node-dev` in watch, `tsc` to `dist/`
- **Version** : 1.0.0

## Default port

`4000` **in code** (`PORT` fallback). In **production** the service listens on **`127.0.0.1:8311`** (systemd unit `hb-api-statbar`, `PORT=8311`). MongoDB is reached at `mongodb://127.0.0.1:27017/statbar` by default.

## Key concepts

- **Bar rating model** : each bar carries a `note` array that must contain **exactly one entry per category** — `DRINK`, `AFFORDABILITY`, `VIBES`, `DECORATION` — each scored `0..5`. A pre-save hook computes `moy`, the average rounded to 2 decimals.
- **Geolocation** : `lat` (`-90..90`) and `lng` (`-180..180`) are required and range-validated; the front places the pin via Photon/OpenStreetMap or manually.
- **Validation at the model** : Mongoose validation errors surface as **HTTP 400** (`ValidationError`), everything else as **500**.
- **Id checks** : route params are validated with `isValidObjectId` — a malformed id returns **400**, an unknown id **404**.
- **No embedded OpenAPI** : unlike HB-Api-Cocktail, this service does not embed a spec; the contract is documented from the routes (see [openapi.yaml](openapi.yaml)).

## Configuration

All configuration comes from **environment variables** (loaded from `.env` via `dotenv`), with defaults applied when absent.

| Variable | Default | Description |
|---|---|---|
| `PORT` | `4000` | HTTP listen port (`8311` in production) |
| `MONGODB_URI` | `mongodb://127.0.0.1:27017/statbar` | MongoDB connection URI |

> `scripts/migrateToAtlas.ts` copies collections from a source MongoDB to a target (`SOURCE_URI` / `TARGET_URI`); it is not part of the standalone local runtime.

## Endpoints

The `bars` resource is mounted under the **`/api/bars`** prefix; `/health` sits at the root.

| Method | Path | Description | Responses |
|---|---|---|---|
| GET | `/health` | Liveness + DB state | 200 `{status, service, db}` |
| POST | `/api/bars` | Create a bar | 201 `Bar`, 400 |
| GET | `/api/bars` | List bars (newest first) | 200 `Bar[]` |
| GET | `/api/bars/{id}` | Get a bar | 200 `Bar`, 400, 404 |
| PUT | `/api/bars/{id}` | Update a bar (partial fields) | 200 `Bar`, 400, 404 |
| DELETE | `/api/bars/{id}` | Delete a bar | 204, 400, 404 |

## `Bar` object

| Field | Type | Detail |
|---|---|---|
| `_id` | string (ObjectId) | Identifier |
| `name` | string | Bar name (required, trimmed) |
| `lat` | number | Latitude, `-90..90` |
| `lng` | number | Longitude, `-180..180` |
| `note` | `Note[]` | Exactly one entry per category |
| `moy` | number | Average of the note values, 2 decimals (computed) |
| `createdAt` / `updatedAt` | string (ISO) | Timestamps |

`Note` : `category` (`DRINK` \| `AFFORDABILITY` \| `VIBES` \| `DECORATION`), `value` (number `0..5`).

## Error format

Errors return a JSON body `{ "error": "<message>" }`. Mongoose `ValidationError` maps to **400**; an invalid/unknown id to **400/404**; anything else to **500**.

## Deployment

Deployed via **`hb-deploy-node hb-api-statbar`** to **`/opt/hobbies/hb-api-statbar`** and run as a **systemd unit** under WSL. CI extends the shared `QVL-ToolBox/PipeLine` Node template: `lint` runs `tsc --noEmit` (no ESLint/Prettier yet), tests are disabled (no suite), `build` produces `dist/`. The pipeline runs on the group runner **`qvl-hobbies-wsl`**.

## OpenAPI specification

See [openapi.yaml](openapi.yaml) (written from the routes; the service does not embed a spec).

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

Service Node/TypeScript du domaine **Hobbies** qui alimente **StatBar**, l'app pour noter, cartographier et classer des bars. Il expose un petit CRUD REST sur la ressource `bars`, adossé à MongoDB via Mongoose. Il n'a **pas d'URL publique propre** : en production, il est consommé par la SPA **HB-Front-StatBar**, qui lui relaie `/api` en même origine.

## Stack

- **Runtime** : Node.js ≥ 20.19
- **HTTP** : Express 5 (`express@^5.2.1`) avec `cors`
- **Langage** : TypeScript (`typescript@^6.0.3`, sortie CommonJS)
- **Base de données** : MongoDB via Mongoose (`mongoose@^9.6.3`)
- **Config** : `dotenv` (`.env`, ignoré par Git)
- **Dev/build** : `ts-node-dev` en watch, `tsc` vers `dist/`
- **Version** : 1.0.0

## Port par défaut

`4000` **dans le code** (repli de `PORT`). En **production**, le service écoute sur **`127.0.0.1:8311`** (unité systemd `hb-api-statbar`, `PORT=8311`). MongoDB est joint par défaut sur `mongodb://127.0.0.1:27017/statbar`.

## Concepts clés

- **Modèle de notation** : chaque bar porte un tableau `note` qui doit contenir **exactement une entrée par catégorie** — `DRINK`, `AFFORDABILITY`, `VIBES`, `DECORATION` — chacune notée `0..5`. Un hook pre-save calcule `moy`, la moyenne arrondie à 2 décimales.
- **Géolocalisation** : `lat` (`-90..90`) et `lng` (`-180..180`) sont obligatoires et validés en plage ; le front place le pin via Photon/OpenStreetMap ou manuellement.
- **Validation au modèle** : les erreurs de validation Mongoose remontent en **HTTP 400** (`ValidationError`), le reste en **500**.
- **Contrôle des id** : les paramètres de route sont validés par `isValidObjectId` — un id mal formé renvoie **400**, un id inconnu **404**.
- **Pas d'OpenAPI embarqué** : contrairement à HB-Api-Cocktail, ce service n'embarque pas de spec ; le contrat est documenté d'après les routes (voir [openapi.yaml](openapi.yaml)).

## Configuration

Toute la configuration provient de **variables d'environnement** (chargées depuis `.env` via `dotenv`), avec des valeurs par défaut si absentes.

| Variable | Défaut | Description |
|---|---|---|
| `PORT` | `4000` | Port d'écoute HTTP (`8311` en production) |
| `MONGODB_URI` | `mongodb://127.0.0.1:27017/statbar` | URI de connexion MongoDB |

> `scripts/migrateToAtlas.ts` copie des collections d'une base MongoDB source vers une cible (`SOURCE_URI` / `TARGET_URI`) ; il ne fait pas partie du fonctionnement local standalone.

## Endpoints

La ressource `bars` est montée sous le préfixe **`/api/bars`** ; `/health` est à la racine.

| Méthode | Chemin | Description | Réponses |
|---|---|---|---|
| GET | `/health` | Vivacité + état de la base | 200 `{status, service, db}` |
| POST | `/api/bars` | Crée un bar | 201 `Bar`, 400 |
| GET | `/api/bars` | Liste les bars (plus récents d'abord) | 200 `Bar[]` |
| GET | `/api/bars/{id}` | Récupère un bar | 200 `Bar`, 400, 404 |
| PUT | `/api/bars/{id}` | Met à jour un bar (champs partiels) | 200 `Bar`, 400, 404 |
| DELETE | `/api/bars/{id}` | Supprime un bar | 204, 400, 404 |

## Objet `Bar`

| Champ | Type | Détail |
|---|---|---|
| `_id` | chaîne (ObjectId) | Identifiant |
| `name` | chaîne | Nom du bar (obligatoire, trimé) |
| `lat` | nombre | Latitude, `-90..90` |
| `lng` | nombre | Longitude, `-180..180` |
| `note` | `Note[]` | Exactement une entrée par catégorie |
| `moy` | nombre | Moyenne des valeurs de note, 2 décimales (calculée) |
| `createdAt` / `updatedAt` | chaîne (ISO) | Horodatages |

`Note` : `category` (`DRINK` \| `AFFORDABILITY` \| `VIBES` \| `DECORATION`), `value` (nombre `0..5`).

## Format d'erreur

Les erreurs renvoient un corps JSON `{ "error": "<message>" }`. Une `ValidationError` Mongoose donne **400** ; un id invalide/inconnu **400/404** ; tout le reste **500**.

## Déploiement

Déployé via **`hb-deploy-node hb-api-statbar`** dans **`/opt/hobbies/hb-api-statbar`** et lancé en **unité systemd** sous WSL. La CI étend le template Node partagé `QVL-ToolBox/PipeLine` : `lint` exécute `tsc --noEmit` (pas encore d'ESLint/Prettier), les tests sont désactivés (aucune suite), `build` produit `dist/`. Le pipeline tourne sur le runner de groupe **`qvl-hobbies-wsl`**.

## Spécification OpenAPI

Voir [openapi.yaml](openapi.yaml) (rédigé d'après les routes ; le service n'embarque pas de spec).

</details>

---

<p align="center"><sub>© QVL — Documentation</sub></p>
