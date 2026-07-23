# HB-Api-Cocktail

<p align="center">
  <em>Cocktails API of the QVL Hobbies domain — public read-only, local-only writes.</em><br>
  <em>API cocktails du domaine QVL Hobbies — lecture publique, écriture locale uniquement.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

Standalone Go service of the **Hobbies** domain serving cocktail recipes. It exposes a **public read-only** surface (list, search by ingredients, detail, reference data, images) plus a **non-public local write** endpoint, reachable only over loopback (`127.0.0.1`) and protected by a bearer token. Recipe text content is in **English**.

Unlike the CustHome services, this API is **not fronted by a gateway**: it is a single local process, bound to loopback by default.

## Stack

- **Language** : Go 1.26.3
- **HTTP** : Go standard library (`net/http`), no web framework
- **Database** : SQLite via the pure-Go driver `modernc.org/sqlite` (no cgo)
- **Binary storage** : local filesystem (images directory)
- **Local write auth** : loopback origin + bearer token, compared in constant time (`crypto/subtle`)
- **API contract** : spec-first OpenAPI 3.0.3, embedded (`//go:embed`) and served, with a Redoc HTML page
- **Version** : 1.0.0

## Default port

`127.0.0.1:8080` (`BIND_ADDR` = `127.0.0.1`, `PORT` = `8080`). The service binds to loopback by default, which already restricts exposure of the write endpoint.

## Key concepts

- **Public read, local write** : all read routes are public; `POST /cocktails` is **mounted only if `LOCAL_WRITE_TOKEN` is set** and remains reserved to authenticated loopback calls.
- **Ingredient search** : `GET /cocktails/search` accepts a comma-separated `ingredients` list and a `match` strategy — `all` (default) requires every ingredient, `any` requires at least one. An unknown ingredient simply narrows the matches (down to an empty list), never an error.
- **Images served by name** : images are exposed via `GET /images/{name}`. The API returns only the image **file name** (field `image`) — **never** the on-disk path.
- **SQLite schema** : `cocktails`, `ingredients`, `cocktail_ingredients` (many-to-many link with `quantity`/`unit`), `cocktail_tags` (one tag = one row). Links are `ON DELETE CASCADE`. The schema is applied idempotently at startup.
- **No config file loading** : the service reads only its process environment (no dotenv loader). Variables must be injected into the environment before launch.

## Configuration

All configuration comes from **environment variables**, with defaults applied when absent. There is no config file and no automatic `.env` loading.

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | HTTP listen port |
| `BIND_ADDR` | `127.0.0.1` | Listen address (loopback by default) |
| `DB_PATH` | `./data/cocktail.db` | SQLite file path (parent dir created if needed) |
| `IMAGES_DIR` | `./data/images` | Images directory |
| `LOCAL_WRITE_TOKEN` | *(empty)* | Bearer token for the local write endpoint. Secret: never committed nor logged. If empty, `POST /cocktails` is **not mounted**. |

## Endpoints

The full v1 surface described in `openapi.yaml` is implemented. Read routes are public; the write route is loopback + token only.

| Method | Path | Auth | Description | Responses |
|---|---|---|---|---|
| GET | `/cocktails` | public | Paginated, filterable list (`category`, `strength`, `alcoholic`, `season`, `tag`, `limit`, `offset`) | 200 `CocktailList`, 400, 500 |
| GET | `/cocktails/search` | public | Search by ingredients (`ingredients`, `match=all\|any`) | 200 `CocktailList`, 400, 500 |
| GET | `/cocktails/{id}` | public | Cocktail detail | 200 `Cocktail`, 400, 404, 500 |
| GET | `/ingredients` | public | Ingredient reference data (`{id, name}` array) | 200, 500 |
| GET | `/categories` | public | Distinct categories (string array) | 200, 500 |
| GET | `/tags` | public | Distinct tags (string array) | 200, 500 |
| GET | `/images/{name}` | public | Cocktail image | 200 (`image/*`), 404, 500 |
| POST | `/cocktails` | local (loopback + bearer) | Create a recipe and upload its image (multipart) | 201 `Cocktail`, 400, 401, 403, 413, 500 |
| GET | `/health` | none | Liveness probe | 200 `{"status":"ok"}` |
| GET | `/openapi.yaml` | none | Raw OpenAPI contract | 200 (`application/yaml`) |
| GET | `/docs` | none | Redoc documentation | 200 (`text/html`) |

## Error format

Every error returns a uniform JSON body with a stable machine-readable `error` code and a human `message` :

```json
{ "error": "not_found", "message": "cocktail not found" }
```

| `error` | Status | When it occurs |
|---|---|---|
| `bad_request` | 400 | Invalid pagination/filter parameter, malformed multipart body, missing `cocktail` part, validation failure, unsupported image type |
| `unauthorized` | 401 | Local write: bearer missing or not matching `LOCAL_WRITE_TOKEN` |
| `forbidden` | 403 | Local write: request from a non-loopback origin |
| `not_found` | 404 | Cocktail not found, or invalid image name / missing file |
| `payload_too_large` | 413 | Image over 2 MB, or overall multipart body too large |
| `internal_error` | 500 | Internal error (database, I/O) |

## `Cocktail` object (DTO)

API objects are distinct from the SQL schema. The storage path is never exposed: only the image **file name** is returned, via `image`.

| Field | Type | Detail |
|---|---|---|
| `id` | integer | Identifier |
| `name` | string | Recipe name |
| `instructions` | string | Preparation |
| `glass` | string | Glass type |
| `category` | string | Category |
| `strength` | string | Strength |
| `alcoholic` | boolean | Alcoholic or not |
| `season` | string | Season (may be empty) |
| `image` | string | Image file name (empty if none), used with `GET /images/{name}` — never the disk path |
| `tags` | string array | Tags, alphabetically sorted |
| `ingredients` | `CocktailIngredient` array | Ingredients, sorted by name |

`CocktailIngredient` : `name` (string), `quantity` (string, may be empty), `unit` (string, may be empty).

`CocktailList` (paginated envelope returned by `/cocktails` and `/cocktails/search`) : `items` (`Cocktail` array), `total` (integer, total matching the filter), `limit` (integer, applied page size — default `20`, bounds `1..100`), `offset` (integer, applied offset — default `0`, bounds `0..100000`).

Reference endpoints return arrays directly: `/ingredients` an array of `{id, name}`, `/categories` and `/tags` string arrays.

## Versioning

The contract version is **1.0.0**. The v1 surface is exposed **at the root** (no version prefix in paths, unlike the CustHome services under `/v1`). Operational routes (`/health`, `/openapi.yaml`, `/docs`) are not versioned.

## OpenAPI specification

See [openapi.yaml](openapi.yaml).

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

Service Go autonome du domaine **Hobbies** dédié aux recettes de cocktails. Il expose une surface **publique en lecture seule** (liste, recherche par ingrédients, détail, référentiels, images) ainsi qu'un endpoint d'**écriture locale non public**, joignable uniquement en loopback (`127.0.0.1`) et protégé par un token bearer. Le contenu textuel des recettes est en **anglais**.

Contrairement aux services CustHome, cette API n'est **pas exposée derrière une gateway** : c'est un unique process local, lié au loopback par défaut.

## Stack

- **Langage** : Go 1.26.3
- **HTTP** : bibliothèque standard Go (`net/http`), sans framework web
- **Base de données** : SQLite via le driver pure-Go `modernc.org/sqlite` (sans cgo)
- **Stockage binaire** : système de fichiers local (dossier d'images)
- **Auth écriture locale** : origine loopback + token bearer, comparé en temps constant (`crypto/subtle`)
- **Contrat d'API** : OpenAPI 3.0.3 spec-first, embarqué (`//go:embed`) et servi, avec une page Redoc
- **Version** : 1.0.0

## Port par défaut

`127.0.0.1:8080` (`BIND_ADDR` = `127.0.0.1`, `PORT` = `8080`). Le service se lie au loopback par défaut, ce qui restreint déjà l'exposition de l'endpoint d'écriture.

## Concepts clés

- **Lecture publique, écriture locale** : toutes les routes de lecture sont publiques ; `POST /cocktails` n'est **monté que si `LOCAL_WRITE_TOKEN` est défini** et reste réservé aux appels loopback authentifiés.
- **Recherche par ingrédients** : `GET /cocktails/search` accepte une liste `ingredients` séparée par des virgules et une stratégie `match` — `all` (défaut) exige tous les ingrédients, `any` en exige au moins un. Un ingrédient inconnu réduit simplement les correspondances (jusqu'à une liste vide), sans erreur.
- **Images servies par nom** : les images sont exposées via `GET /images/{name}`. L'API ne renvoie que le **nom de fichier** de l'image (champ `image`) — **jamais** le chemin sur disque.
- **Schéma SQLite** : `cocktails`, `ingredients`, `cocktail_ingredients` (liaison many-to-many avec `quantity`/`unit`), `cocktail_tags` (un tag = une ligne). Les liaisons sont en `ON DELETE CASCADE`. Le schéma est appliqué de façon idempotente au démarrage.
- **Aucun chargement de fichier de config** : le service lit uniquement l'environnement de son process (pas de loader dotenv). Les variables doivent être injectées dans l'environnement avant le lancement.

## Configuration

Toute la configuration provient de **variables d'environnement**, avec des valeurs par défaut si absentes. Il n'y a pas de fichier de config ni de chargement automatique de `.env`.

| Variable | Défaut | Description |
|---|---|---|
| `PORT` | `8080` | Port d'écoute HTTP |
| `BIND_ADDR` | `127.0.0.1` | Adresse d'écoute (loopback par défaut) |
| `DB_PATH` | `./data/cocktail.db` | Chemin du fichier SQLite (dossier parent créé si besoin) |
| `IMAGES_DIR` | `./data/images` | Dossier des images |
| `LOCAL_WRITE_TOKEN` | *(vide)* | Token bearer de l'endpoint d'écriture locale. Secret : ni commité ni loggé. Si vide, `POST /cocktails` n'est **pas monté**. |

## Endpoints

Toute la surface v1 décrite dans `openapi.yaml` est implémentée. Les routes de lecture sont publiques ; la route d'écriture est loopback + token uniquement.

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/cocktails` | public | Liste paginée et filtrable (`category`, `strength`, `alcoholic`, `season`, `tag`, `limit`, `offset`) | 200 `CocktailList`, 400, 500 |
| GET | `/cocktails/search` | public | Recherche par ingrédients (`ingredients`, `match=all\|any`) | 200 `CocktailList`, 400, 500 |
| GET | `/cocktails/{id}` | public | Détail d'un cocktail | 200 `Cocktail`, 400, 404, 500 |
| GET | `/ingredients` | public | Référentiel des ingrédients (tableau `{id, name}`) | 200, 500 |
| GET | `/categories` | public | Catégories distinctes (tableau de chaînes) | 200, 500 |
| GET | `/tags` | public | Tags distincts (tableau de chaînes) | 200, 500 |
| GET | `/images/{name}` | public | Image d'un cocktail | 200 (`image/*`), 404, 500 |
| POST | `/cocktails` | local (loopback + bearer) | Ajout d'une recette et upload d'image (multipart) | 201 `Cocktail`, 400, 401, 403, 413, 500 |
| GET | `/health` | non | Sonde de vivacité | 200 `{"status":"ok"}` |
| GET | `/openapi.yaml` | non | Contrat OpenAPI brut | 200 (`application/yaml`) |
| GET | `/docs` | non | Documentation Redoc | 200 (`text/html`) |

## Format d'erreur

Toute erreur renvoie un corps JSON uniforme avec un code `error` stable lisible par machine et un `message` humain :

```json
{ "error": "not_found", "message": "cocktail not found" }
```

| `error` | Statut | Quand il survient |
|---|---|---|
| `bad_request` | 400 | Paramètre de pagination/filtre invalide, corps multipart mal formé, part `cocktail` manquante, échec de validation, type d'image non supporté |
| `unauthorized` | 401 | Écriture locale : bearer absent ou ne correspondant pas à `LOCAL_WRITE_TOKEN` |
| `forbidden` | 403 | Écriture locale : requête émise depuis une origine non loopback |
| `not_found` | 404 | Cocktail introuvable, ou nom d'image invalide / fichier absent |
| `payload_too_large` | 413 | Image au-delà de 2 Mo, ou corps multipart global trop volumineux |
| `internal_error` | 500 | Erreur interne (base de données, I/O) |

## Objet `Cocktail` (DTO)

Les objets renvoyés par l'API sont distincts du schéma SQL. Le chemin de stockage n'est jamais exposé : seul le **nom de fichier** de l'image est renvoyé, via `image`.

| Champ | Type | Détail |
|---|---|---|
| `id` | entier | Identifiant |
| `name` | chaîne | Nom de la recette |
| `instructions` | chaîne | Préparation |
| `glass` | chaîne | Type de verre |
| `category` | chaîne | Catégorie |
| `strength` | chaîne | Force |
| `alcoholic` | booléen | Alcoolisé ou non |
| `season` | chaîne | Saison (peut être vide) |
| `image` | chaîne | Nom de fichier image (vide si absente), à utiliser sur `GET /images/{name}` — jamais le chemin disque |
| `tags` | tableau de chaînes | Tags triés par ordre alphabétique |
| `ingredients` | tableau de `CocktailIngredient` | Ingrédients triés par nom |

`CocktailIngredient` : `name` (chaîne), `quantity` (chaîne, peut être vide), `unit` (chaîne, peut être vide).

`CocktailList` (enveloppe paginée renvoyée par `/cocktails` et `/cocktails/search`) : `items` (tableau de `Cocktail`), `total` (entier, total correspondant au filtre), `limit` (entier, taille de page appliquée — défaut `20`, bornes `1..100`), `offset` (entier, décalage appliqué — défaut `0`, bornes `0..100000`).

Les référentiels renvoient directement des tableaux : `/ingredients` un tableau de `{id, name}`, `/categories` et `/tags` des tableaux de chaînes.

## Versionnement

La version du contrat est **1.0.0**. La surface v1 est exposée **à la racine** (pas de préfixe de version dans les chemins, contrairement aux services CustHome sous `/v1`). Les routes opérationnelles (`/health`, `/openapi.yaml`, `/docs`) ne sont pas versionnées.

## Spécification OpenAPI

Voir [openapi.yaml](openapi.yaml).

</details>
