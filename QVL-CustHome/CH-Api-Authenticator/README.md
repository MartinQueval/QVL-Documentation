# CH-Api-Authenticator

Microservice d'**authentification multi-portail** de l'écosystème CustHome. Il gère les comptes, les rôles par portail, l'émission et la validation des JWT, les sessions (refresh tokens), la réinitialisation de mot de passe et l'administration des utilisateurs.

Il est appelé par [CH-Api-GateWay](../CH-Api-GateWay/README.md) :
- pour **chaque requête protégée** via `GET /validate` (validation du JWT sans I/O, budget < 100 ms) ;
- pour les routes publiques `/api/auth/*` (login, register…) et d'administration `/api/admin/*`, proxifiées.

## Stack

- **Langage** : Rust (edition 2024)
- **Framework HTTP** : Axum 0.8
- **Base de données** : MongoDB (driver `mongodb` 3)
- **Sécurité** : Argon2id (hachage des mots de passe), JWT HS256 (`jsonwebtoken`), rate limiting (`governor`)
- **Version** : 0.1.0

## Port par défaut

`8181` (clé `server.port` de `config.toml`, surchargée par la variable `PORT`).

## Concepts clés

- **Rôles par portail** : un utilisateur porte une liste de rôles ; le rôle nommé comme un portail (`admin`, `drive`, `home`, `budgy`) donne accès au portail correspondant. La Gateway transmet le portail visé via le header `X-Portal` ; l'Authenticator vérifie que ce portail est connu au `/validate`.
- **Whitelist IP par utilisateur** : `whitelist_only` + `allowed_ips` (IP ou CIDR), vérifiée au login. Quand elle est active, le token embarque l'IP (claim `ip`), contrôlée au `/validate` via `X-Client-IP`.
- **JWT stateless HS256** (TTL 15 min) posé en cookie `HttpOnly` (`ch_token`) au login, accompagné d'un **refresh token** rotatif (cookie `ch_refresh`, TTL 7 jours) avec détection de réutilisation (révocation de la famille).
- **Statuts de compte** : `pending_validation`, `active`, `disabled`.
- **CGU** : l'inscription exige l'acceptation de la version courante des conditions générales.

## Configuration

Configuration non sensible dans `config.toml`, surchargeable par variables d'environnement préfixées `CH__` (ex. `CH__SERVER__PORT=9000`). Les secrets viennent exclusivement de l'environnement.

### config.toml

| Section / clé | Défaut | Description |
|---|---|---|
| `server.port` | `8181` | Port d'écoute |
| `server.log_level` | `INFO` | Verbosité (`DEBUG`, `INFO`, `WARN`, `ERROR`) |
| `token.ttl_minutes` | `15` | Durée de vie de l'access token JWT |
| `token.cookie_name` | `ch_token` | Cookie HttpOnly de l'access token |
| `token.cookie_secure` | `false` | `true` forcé en production |
| `token.refresh_ttl_days` | `7` | Durée de vie du refresh token |
| `token.refresh_cookie_name` | `ch_refresh` | Cookie HttpOnly du refresh token |
| `registration.default_roles` | `[]` | Rôles attribués automatiquement à l'inscription |
| `missive.url` | `http://localhost:8184` | Service d'envoi des emails de reset |
| `password_reset.url` | `http://localhost:3200/reset-password` | Page front consommant le token de reset |
| `password_reset.ttl_minutes` | `15` | Durée de vie du token de reset |
| `relay.enabled` | `false` | Publication des événements sur le bus Relay (MQTT 5) |
| `relay.host` / `relay.port` | `127.0.0.1` / `1883` | Broker Relay |

### Variables d'environnement (secrets et surcharges)

| Variable | Requis | Description |
|---|---|---|
| `JWT_SECRET` | oui | Secret HS256 (≥ 32 octets) |
| `INTERNAL_API_SECRET` | oui | Secret inter-services (≥ 32 octets, distinct de `JWT_SECRET`) |
| `MONGO_URI` | oui | Connexion MongoDB |
| `MISSIVE_API_SECRET` | oui | Secret partagé avec Missive (≥ 32 octets) |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | non | Seed du premier super-admin au démarrage (idempotent) |
| `PORT` | non | Surcharge `server.port` |
| `MISSIVE_URL` | non | Surcharge `missive.url` |
| `JWT_ISSUER`, `JWT_AUDIENCE_DRIVE`, `JWT_AUDIENCE_BUDGY` | non | Contrat JWT (issuer / audiences) |
| `RELAY_JWT_PRIVATE_KEY` | si `relay.enabled` | Clé privée RSA pour signer le JWT présenté à Relay |

Rate limiting configurable : `AUTH_RATE_LIMIT_LOGIN_MAX` (5), `AUTH_RATE_LIMIT_LOGIN_WINDOW_SECS` (300), `AUTH_RATE_LIMIT_FORGOT_MAX` (3), `AUTH_RATE_LIMIT_FORGOT_WINDOW_SECS` (900), `AUTH_RATE_LIMIT_REFRESH_MAX` (30), `AUTH_RATE_LIMIT_REFRESH_WINDOW_SECS` (60).

## Endpoints

Les routes applicatives sont exposées **sous le préfixe `/v1`** et, pour compatibilité, **également sans préfixe**. `X-Portal admin` désigne les routes réservées aux administrateurs (rôle `admin`). Via la Gateway, elles sont atteintes sous `/api/auth/...` (public) ou `/api/admin/...` (admin).

### Authentification et session

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| POST | `/register` | non | Inscription (Argon2id, CGU requises) | 201, 400, 403, 409, 422 |
| POST | `/login` | non | Connexion → JWT + cookies HttpOnly | 200, 400, 401, 403, 429 |
| POST | `/refresh` | cookie `ch_refresh` | Rotation du refresh token → nouvelle session | 200, 401, 429 |
| POST | `/logout` | cookie `ch_refresh` | Déconnexion (révocation de la famille de tokens) | 200 |
| GET | `/validate` | Bearer + `X-Portal` | Validation du token pour la Gateway | 200, 401, 403 |
| PUT | `/password` | Bearer | Changement de mot de passe (mot de passe actuel requis) | 200, 400, 401 |
| POST | `/password/forgot` | non | Demande de lien de réinitialisation (anti-énumération : 202 systématique) | 202, 400, 429 |
| POST | `/password/reset` | non | Réinitialisation via token one-time | 200, 400 |

### Profil courant

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/me` | Bearer | Profil de l'utilisateur courant | 200, 401 |
| PUT | `/me` | Bearer | Mise à jour de l'email | 200, 400, 401, 409 |

### Administration (rôle `admin`)

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/users` | admin | Liste paginée des comptes (filtres `email`, `status`) | 200, 401, 403 |
| GET | `/users/pending` | admin | Comptes en attente de validation | 200, 401, 403 |
| GET | `/users/{id}` | admin | Détail d'un compte | 200, 401, 403, 404 |
| PUT | `/users/{id}` | admin | Mise à jour nom + email | 200, 400, 404, 409 |
| DELETE | `/users/{id}` | admin | Suppression d'un compte (publie un événement) | 204, 404 |
| PUT | `/users/{id}/status` | admin | Changement de statut | 200, 400, 404 |
| PUT | `/users/{id}/password` | admin | Réinitialisation du mot de passe | 204, 400, 404 |
| PUT | `/users/{id}/roles` | admin | Remplacement des rôles (validés contre le catalogue) | 200, 400, 404 |
| PUT | `/users/{id}/whitelist` | admin | Configuration de la whitelist IP | 200, 400, 404 |
| GET | `/analytics/traffic` | admin | Trafic par portail (query `period`: day/week/month/year) | 200, 401, 403 |
| GET | `/settings/registration` | Bearer | État de l'inscription publique | 200 |
| PUT | `/settings/registration` | admin | Activation/désactivation de l'inscription | 200 |
| GET | `/roles` | admin | Catalogue des rôles | 200, 401, 403 |
| POST | `/roles` | admin | Création d'un sous-rôle de portail | 201, 400, 409 |
| DELETE | `/roles/{id}` | admin | Suppression d'un sous-rôle (rôle portail interdit) | 204, 403, 404 |

### Opérationnel et interne (non versionnés)

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/health` | non | État du service (+ ping MongoDB) | 200 |
| GET | `/ping` | non | Route témoin | 200 |
| POST | `/internal/users/resolve` | `x-internal-secret` | Résolution `id → {name, email}` (inter-services) | 200, 403 |

## Format d'erreur

```json
{ "error": "bad_request", "message": "requête invalide : format d'email invalide" }
```

| `error` | Statut | Cas |
|---|---|---|
| `bad_request` | 400 | Validation du corps |
| `unauthorized` | 401 | Identifiants ou token invalides |
| `forbidden` | 403 | Accès refusé (rôle, portail, compte pending/disabled/device) |
| `not_found` | 404 | Ressource inconnue |
| `conflict` | 409 | Email ou rôle déjà utilisé |
| `terms_not_accepted` / `terms_version_mismatch` | 422 | CGU non acceptées / version obsolète |
| `too_many_requests` | 429 | Rate limit (header `Retry-After`) |
| `internal_error` | 500 | Erreur interne |

## Pagination

Les listes de comptes utilisent les query params `page` (défaut 1) et `limit` (défaut 20, max 100). L'enveloppe de réponse est :

```json
{ "users": [ ... ], "page": 1, "limit": 20, "total": 143 }
```

## Spécification OpenAPI

Voir [openapi.yaml](openapi.yaml).
