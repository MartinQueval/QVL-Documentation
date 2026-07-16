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
- **Version** : 0.1.0

## Port par défaut

`8182` (clé `server.port` de `config.toml`, surchargée par la variable `PORT`).

## Concepts clés

- **Nœuds (`node`)** : un nœud est un dossier (`folder`) ou un fichier (`file`). Chaque utilisateur possède un dossier racine implicite. Les nœuds portent nom, mime, taille, métadonnées média (largeur, hauteur, `taken_at`, vignette) et un statut de corbeille (`trashed`).
- **Isolation par propriétaire** : toutes les opérations sont scoppées au `sub` du JWT (identifiant du compte). L'identité doit être un ObjectId valide.
- **Quotas** : chaque utilisateur Drive a un `quota_bytes` et un `used_bytes`. Le quota par défaut est configurable ; l'upload est refusé au-delà (`413 quota_exceeded`).
- **Upload par chunks** : session ouverte (`POST /uploads`), envoi des chunks (`PUT /uploads/{id}/chunks/{index}`), finalisation (`POST /uploads/{id}/complete`). Les sessions ont un TTL (24 h) et réservent le quota. États : `open`, `completing`, `completed`, `aborted`.
- **Corbeille** : mise en corbeille (`trash`), restauration (`restore`), purge (`purge_trash` ou suppression définitive d'un nœud).
- **Sécurité de service** : `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff`, neutralisation des types actifs (SVG, HTML, JS…) au téléchargement ; support des requêtes `Range`.
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

| Variable | Requis | Description |
|---|---|---|
| `DATABASE_URL` | oui | Connexion PostgreSQL |
| `JWT_SECRET` | oui | Secret HS256 (≥ 32 octets, identique à l'Authenticator) |
| `INTERNAL_API_SECRET` | oui | Secret inter-services (≥ 32 octets) |
| `PORT` | non | Surcharge `server.port` |
| `AUTH_INTERNAL_URL` | non | Surcharge `auth_internal_url` |
| `JWT_ISSUER`, `JWT_AUDIENCE` | non | Contrat JWT |

## Endpoints

Toutes les routes applicatives sont sous le préfixe **`/v1`**. Auth : JWT (Bearer ou cookie `ch_token`). Via la Gateway : `/api/drive/v1/...`. Limites de corps : upload multipart 256 Mio, chunk 17 Mio, autres appels 1 Mio.

### Fichiers et dossiers

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/me/storage` | JWT | Quota et usage de l'utilisateur | 200, 401 |
| GET | `/files` | JWT | Liste le contenu d'un dossier (query `parent`) + fil d'Ariane | 200, 401, 404 |
| POST | `/files` | JWT | Upload direct multipart (champ `file`, query `parent`) | 201, 400, 401, 413 |
| GET | `/files/{id}/content` | JWT | Téléchargement (supporte `Range`) | 200, 206, 401, 404, 416 |
| GET | `/files/{id}/thumbnail` | JWT | Vignette JPEG (images) | 200, 401, 404 |
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

Les routes applicatives sont exposées sous `/v1`. Les routes opérationnelles (`/health`) ne sont pas versionnées. Voir aussi `API_VERSIONING.md` dans le repo source.

> Note : le document `API_VERSIONING.md` du repo décrit une double exposition (chemins historiques sans préfixe + `/v1`). Dans le code actuel (`src/routes.rs`), seules les routes préfixées `/v1` sont montées, en plus de `/health`.

## Spécification OpenAPI

Voir [openapi.yaml](openapi.yaml).
