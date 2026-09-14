# CH-Api-Drive

Microservice **Drive** de l'écosystème CustHome : stockage de fichiers arborescent (dossiers / fichiers), galerie média (images / vidéos), corbeille, upload direct et upload par chunks (reprise), quotas par utilisateur et administration.

Il est exposé par [CH-Api-GateWay](../CH-Api-GateWay/README.md) sous le préfixe `/api/drive` (auth requise, portail `portail_drive`, corps ≤ 17 Mo, timeout 120 s).

## Stack

- **Langage** : Rust (edition 2024)
- **Framework HTTP** : Axum 0.8 (multipart)
- **Base de données** : PostgreSQL (`sqlx`)
- **Stockage binaire** : système de fichiers local (racine configurable)
- **Média** : `image` (vignettes), `kamadak-exif` (métadonnées)
- **Auth** : JWT HS256 validé localement (secret partagé avec l'Authenticator)
- **Version** : v1.0.0 (tag Git ; `Cargo.toml` porte encore `0.1.0`)

### Dépendances système : `poppler-utils` et `ffmpeg`

Le rendu média s'appuie sur deux outils appelés en sous-processus (dépendances de **runtime** uniquement — la compilation n'en a pas besoin) :

| Outil | Paquet | Usage |
|---|---|---|
| `pdftoppm`, `pdfinfo` | `poppler-utils` | Vignette et aperçu (rendu de page) des **PDF** |
| `ffmpeg` | `ffmpeg` | Vignette des **vidéos** (extraction d'une image à `00:00:01`, repli sur `00:00:00`) |

```bash
sudo apt install poppler-utils ffmpeg
```

Choix assumé plutôt que des bibliothèques natives liées au binaire : aucun moteur de rendu PDF pur Rust n'est mûr, et lier `pdfium`/`mupdf` alourdirait la compilation comme le déploiement pour un besoin marginal.

Si un outil est absent, la génération échoue silencieusement : le fichier est créé normalement et l'interface affiche l'icône de type. Aucun upload n'est jamais mis en échec par l'aperçu.

## Port par défaut

`8182` (clé `server.port` de `config.toml`, surchargée par la variable `PORT`).

## Concepts clés

- **Nœuds (`node`)** : un nœud est un dossier (`folder`) ou un fichier (`file`). Chaque utilisateur possède un dossier racine implicite. Les nœuds portent nom, mime, taille, métadonnées média (largeur, hauteur, `taken_at`, vignette) et un statut de corbeille (`trashed`).
- **Isolation par propriétaire** : toutes les opérations sont scoppées au `sub` du JWT (identifiant du compte). L'identité doit être un ObjectId valide.
- **Quotas** : chaque utilisateur Drive a un `quota_bytes` et un `used_bytes`. Le quota par défaut est configurable ; l'upload est refusé au-delà (`413 quota_exceeded`).
- **Upload par chunks** : session ouverte (`POST /uploads`), envoi des chunks (`PUT /uploads/{id}/chunks/{index}`), finalisation (`POST /uploads/{id}/complete`). Les sessions ont un TTL (24 h) et réservent le quota. États : `open`, `completing`, `completed`, `aborted`. Limites : taille de chunk ≤ **16 Mio** (`MAX_CHUNK_BYTES`), taille déclarée du fichier ≤ **10 Gio** (`MAX_DECLARED_SIZE_BYTES`).
- **Corbeille** : mise en corbeille (`trash`), restauration (`restore`), purge (`purge_trash` ou suppression définitive d'un nœud).
- **Vignettes** : générées à l'upload — par les deux chemins, simple et par chunks — pour les images (via `image`), les PDF (première page, via `pdftoppm`) et les vidéos (via `ffmpeg`). Exposées par `GET /files/{id}/thumbnail` et signalées par `has_thumbnail`. L'échec de génération n'est jamais bloquant.
- **Aperçu PDF** : rendu de page à la demande. `GET /files/{id}/preview` renvoie le nombre de pages (`pdfinfo`) ; `GET /files/{id}/preview/{page}` rasterise la page en JPEG (`pdftoppm`), avec cache sur disque. Réservé aux fichiers `application/pdf` (404 sinon).
- **Sécurité de service** : `X-Content-Type-Options: nosniff`, neutralisation des types actifs (SVG, HTML, JS…) au téléchargement ; support des requêtes `Range` (206 Partial Content). Le `Content-Disposition` vaut **`inline` pour une liste étroite** (PDF, images bitmap et **vidéos** `mp4`/`webm`/`ogg`, servies en lecture directe), afin de permettre la prévisualisation dans le navigateur, et **`attachment` pour tout le reste**. La décision se prend sur le type *servi*, après neutralisation : un format actif ne peut donc pas être rendu dans l'origine de l'application.
- **Administration** : rôle `drive_admin` requis. La résolution nom/email des propriétaires s'appuie sur `POST /internal/users/resolve` de l'Authenticator.
- **Événements** : publication d'un événement « fichier téléversé » sur le bus Relay (best-effort).

## Configuration

Configuration non sensible dans `config.toml`, surchargeable par variables `CH__`. Secrets par environnement.

### config.toml

| Section / clé | Défaut | Description |
|---|---|---|
| `server.port` | `8182` | Port d'écoute |
| `server.log_level` | `INFO` | Verbosité |
| `storage.root` | `./Drive` | Racine du stockage binaire |
| `storage.default_quota_bytes` | `16106127360` (15 Gio) | Quota par défaut par utilisateur |
| `token.cookie_name` | `ch_token` | Cookie d'access token accepté |
| `token.issuer` | `ch-api-authenticator` | Issuer JWT attendu |
| `token.audience` | `ch-api-drive` | Audience JWT attendue |
| `upload_gc.interval_secs` | `300` | Intervalle du GC des sessions d'upload |
| `upload_gc.batch_size` | `100` | Taille de lot du GC |
| `auth_internal_url` | `http://localhost:8181` | Base de l'Authenticator (résolution d'identités) |

### Variables d'environnement

Les variables **requises** sont vérifiées au démarrage : si l'une manque (ou est vide), le service **s'arrête immédiatement** (`exit(1)`).

| Variable | Requis | Description |
|---|---|---|
| `DATABASE_URL` | oui | Connexion PostgreSQL |
| `JWT_SECRET` | oui | Secret HS256 (≥ 32 octets, identique à l'Authenticator) |
| `INTERNAL_API_SECRET` | oui | Secret inter-services (≥ 32 octets) |
| `RELAY_MQTT_URL` | oui | URL du broker Relay (schéma **non chiffré** requis : `mqtt://` ou `tcp://` ; `mqtts`/`ssl`/`tls` rejetés) |
| `RELAY_SERVICE_IDENTITY` | oui | Identité (client id) du service au CONNECT MQTT |
| `RELAY_SERVICE_TOKEN` | oui | JWT de service présenté comme mot de passe MQTT |
| `RELAY_UPLOAD_TOPIC_TEMPLATE` | oui | Gabarit du topic « fichier téléversé » (doit contenir `{owner_id}` et `{file_id}`) |
| `PORT` | non | Surcharge `server.port` |
| `AUTH_INTERNAL_URL` | non | Surcharge `auth_internal_url` |
| `JWT_ISSUER`, `JWT_AUDIENCE` | non | Contrat JWT |

## Endpoints

Les routes applicatives sont servies **à la racine, sans préfixe de version** (`/v1` abandonné le 2026-07-22, commit `bbe5b09`). Auth : JWT (Bearer ou cookie `ch_token`). Via la Gateway : `/api/drive/...` (ex. `GET /api/drive/files`). Limites de corps : upload multipart 256 Mio, chunk 17 Mio, autres appels 1 Mio.

### Fichiers et dossiers

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/me/storage` | JWT | Quota et usage de l'utilisateur | 200, 401 |
| GET | `/files` | JWT | Liste le contenu d'un dossier (query `parent`) + fil d'Ariane | 200, 401, 404 |
| POST | `/files` | JWT | Upload direct multipart (champ `file`, query `parent`) | 201, 400, 401, 413 |
| GET | `/files/{id}/content` | JWT | Téléchargement (supporte `Range`) | 200, 206, 401, 404, 416 |
| GET | `/files/{id}/thumbnail` | JWT | Vignette JPEG (images, PDF, vidéos) | 200, 401, 404 |
| GET | `/files/{id}/preview` | JWT | Nombre de pages d'un PDF (`{ "pages": N }`) | 200, 401, 404 |
| GET | `/files/{id}/preview/{page}` | JWT | Rendu JPEG d'une page de PDF | 200, 401, 404 |
| GET | `/gallery` | JWT | Liste des médias (galerie) | 200, 401 |
| GET | `/search` | JWT | Recherche par nom (query `q`) | 200, 401 |
| GET | `/duplicates` | JWT | Fichiers en doublon (même hash) | 200, 401 |
| POST | `/folders` | JWT | Création d'un dossier | 201, 400, 401, 409 |
| PATCH | `/nodes/{id}` | JWT | Renommer et/ou déplacer un nœud | 200, 400, 401, 404 |
| DELETE | `/nodes/{id}` | JWT | Suppression définitive d'un nœud | 204, 401, 403, 404 |
| POST | `/nodes/{id}/trash` | JWT | Mise en corbeille (sous-arbre) | 204, 401, 403, 404 |
| POST | `/nodes/{id}/restore` | JWT | Restauration depuis la corbeille | 200, 401, 404 |
| GET | `/trash` | JWT | Contenu de la corbeille | 200, 401 |
| POST | `/trash/purge` | JWT | Vidage définitif de la corbeille | 204, 401 |

### Upload par chunks

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| POST | `/uploads` | JWT | Ouverture d'une session d'upload (réserve le quota) | 201, 400, 401, 413 |
| GET | `/uploads/{id}` | JWT | État d'une session | 200, 401, 404 |
| PUT | `/uploads/{id}/chunks/{index}` | JWT | Envoi d'un chunk (corps binaire) | 200, 400, 401, 404, 409, 413 |
| POST | `/uploads/{id}/complete` | JWT | Finalisation → matérialisation du fichier | 201, 401, 404, 409 |
| DELETE | `/uploads/{id}` | JWT | Annulation d'une session | 204, 401, 404, 409 |

### Administration (rôle `drive_admin`)

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/admin/users` | drive_admin | Liste des utilisateurs Drive (quota / usage, nom / email résolus) | 200, 401, 403 |
| PATCH | `/admin/users/{id}` | drive_admin | Modification du quota | 200, 400, 403, 404 |
| POST | `/admin/users/{id}/recompute` | drive_admin | Recalcul de `used_bytes` | 200, 403, 404 |

### Opérationnel

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/health` | non | État du service | 200 |

## Format d'erreur

```json
{ "error": "not_found", "message": "Fichier introuvable." }
```

| `error` | Statut | Cas |
|---|---|---|
| `bad_request` | 400 | Validation (nom, mime, chunk hors limites…) |
| `unauthorized` | 401 | Token invalide ou absent |
| `forbidden` | 403 | Action interdite (racine, rôle) |
| `not_found` | 404 | Nœud ou session introuvable |
| `conflict` | 409 | Nom déjà existant, état de session invalide |
| `quota_exceeded` | 413 | Quota de stockage dépassé |
| `payload_too_large` | 413 | Corps de requête au-delà de la limite |
| `internal_error` | 500 | Erreur interne |

## Objet `Node` (DTO)

Un nœud renvoyé par l'API porte : `id`, `parent_id`, `kind` (`folder`/`file`), `name`, `mime`, `size_bytes`, `is_media`, `media_type` (`image`/`video`), `width`, `height`, `duration_ms`, `has_thumbnail`, `taken_at`, `trashed`, `created_at`, `updated_at`.

## Versionnement

Drive **n'expose plus de préfixe de version** : le versionnement d'URL `/v1` a été abandonné le 2026-07-22 (commit `bbe5b09`, « refactor(drive): abandonne le versionnement d'URL /v1 »). Les routes applicatives sont servies à la racine (`/files`, `/gallery`, `/uploads`…), les routes opérationnelles (`/health`) également. Voir `API_VERSIONING.md` dans le repo source.

> Divergence assumée avec CH-Api-Authenticator (qui conserve `/v1`) : la source de vérité des routes est `src/routes.rs`.

## Spécification OpenAPI

Voir [openapi.yaml](openapi.yaml).
