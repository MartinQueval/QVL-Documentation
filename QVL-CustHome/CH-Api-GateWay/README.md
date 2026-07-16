# CH-Api-GateWay

**API Gateway** de la plateforme CustHome : point d'entrée HTTP unique qui route les requêtes vers les microservices back, avec authentification déléguée, limitation de trafic, CORS, limite de taille de corps et observabilité.

## Stack

- **Langage** : Go 1.26 (bibliothèque standard)
- **Dépendances** : `gopkg.in/yaml.v3`, `golang.org/x/time` (rate limiting), `github.com/google/uuid` (correlation ID), `github.com/joho/godotenv`
- **Configuration** : fichier YAML statique, validé strictement au démarrage

## Port par défaut

`8180` (clé `server.port` de `config.yaml`, surchargée par la variable `PORT`).

## Rôle et chaîne de traitement

La Gateway assemble une chaîne de middlewares (`internal/app/app.go`), du plus externe au plus interne :

1. **Correlation ID** — réutilise ou génère `X-Correlation-ID` (UUID v4).
2. **Résolution de l'IP client** — via `trusted_proxies` / `X-Forwarded-For`.
3. **Logs JSON** (`log/slog`) — une ligne par requête.
4. **Rate limiting** — Token Bucket par IP (`/health` exempté), `429` au-delà.
5. **Limite de taille de corps** — `413` au-delà de `max_body_bytes`.
6. **Strip des en-têtes non fiables** — supprime `X-User-Id` / `X-User-Role` entrants (anti-usurpation).
7. **CORS** — preflight `OPTIONS` et en-têtes `Allow-Origin`.
8. **Routage par préfixe** (`http.ServeMux`) — le préfixe le plus long gagne ; `404` sinon.
   - **Auth** (si `require_auth`) : validation du Bearer / cookie auprès de l'Authenticator.
   - **Timeout backend** (`504` au-delà de `timeout_seconds`).
   - **Reverse proxy** vers `destination_url` (avec `strip_prefix` éventuel).

## Configuration

Fichier `config.yaml` (chemin via le flag `-config`, défaut `config.yaml`). Lecture unique au démarrage, validation stricte (les champs YAML inconnus sont rejetés).

### Racine

| Champ | Type | Défaut | Notes |
|---|---|---|---|
| `environment` | string | `development` | `development` ou `production` ; pilote le durcissement CORS |
| `auth_service_url` | string | — | URL de validation des tokens (requis si une route a `require_auth`) |
| `auth_service_timeout_ms` | int | `100` | Timeout de l'appel de validation |
| `auth_cookie_name` | string | `ch_token` | Cookie porteur du token si pas de header Bearer |
| `auth_front_url` | string | — | Page de login (redirection des requêtes HTML non authentifiées) |
| `routes` | []route | — | Au moins une route requise |

### server

| Champ | Type | Défaut | Contraintes |
|---|---|---|---|
| `port` | int | — (requis) | 1–65535 |
| `timeout_seconds` | int | `5` | ≥ 1 |
| `max_body_bytes` | int64 | `10485760` (10 Mio) | ≥ 1 |
| `log_level` | string | `INFO` | `DEBUG`, `INFO`, `WARN`, `ERROR` |
| `rate_limit.enabled` | bool | `false` | — |
| `rate_limit.requests_per_second` | float | — | > 0 si activé |
| `rate_limit.burst` | int | — | ≥ 1 si activé |
| `rate_limit.trusted_proxies` | []string | `[]` | IP ou CIDR valides |
| `cors.allowed_origins` | []string | `[]` | `"*"` rejeté hors `development` |
| `cors.allowed_methods` | []string | `[]` | insensible à la casse |
| `cors.allowed_headers` | []string | `[]` | renvoyés tels quels au preflight |
| `cors.max_age_seconds` | int | `0` | ≥ 0 |

### routes[]

| Champ | Type | Notes |
|---|---|---|
| `path_prefix` | string | doit commencer par `/` ; unique |
| `destination_url` | string | URL http(s) avec hôte |
| `strip_prefix` | bool | retire `path_prefix` avant de proxifier |
| `require_auth` | bool | exige un token valide |
| `portal` | string | requis si `require_auth` ; doit commencer par `portail_` ; transmis en `X-Portal` |
| `max_body_bytes` | int64 | surcharge la limite de corps pour la route |
| `timeout_seconds` | int | surcharge le timeout pour la route |

### Variables d'environnement

| Variable | Effet |
|---|---|
| `GATEWAY_ENV` | Surcharge `environment` |
| `PORT` | Surcharge `server.port` |
| `AUTH_SERVICE_URL` | Surcharge `auth_service_url` |
| `AUTH_FRONT_URL` | Surcharge `auth_front_url` |
| `CORS_ALLOWED_ORIGINS` | Surcharge `cors.allowed_origins` (liste séparée par des virgules) |

## Mapping de routage (config.yaml courant)

`auth_service_url` = `http://localhost:8181/validate` · `auth_cookie_name` = `ch_token` · `auth_front_url` = `http://localhost:3200/login`.

| Préfixe | Destination | strip_prefix | require_auth | portal | Particularités |
|---|---|---|---|---|---|
| `/api/auth` | `http://localhost:8181` | oui | non | `portail_home` | Routes publiques Authenticator |
| `/api/admin` | `http://localhost:8181` | oui | oui | `portail_admin` | Administration Authenticator |
| `/api/drive` | `http://localhost:8182` | oui | oui | `portail_drive` | `max_body_bytes` 17825792, `timeout_seconds` 120 |
| `/api/budgy` | `http://localhost:8183` | oui | oui | `portail_budgy` | — |

CORS autorisé pour les origines `http://localhost:3200` à `3203` (portails).

Exemples de réécriture : `POST /api/auth/login` → `POST http://localhost:8181/login` ; `GET /api/drive/v1/files` → `GET http://localhost:8182/v1/files`.

## Authentification déléguée

Pour les routes `require_auth: true` (`internal/middleware/auth.go`) :

1. Les en-têtes entrants `X-User-Id` / `X-User-Role` sont supprimés.
2. Le token est extrait de `Authorization: Bearer <token>` ou du cookie `ch_token` → `401` s'il est absent (redirection vers `auth_front_url` pour une requête HTML).
3. La Gateway appelle `GET auth_service_url` avec le token, le header `X-Portal` (portail de la route), `X-Client-IP` et `X-Correlation-ID`.
4. Selon la réponse :

| Réponse du service d'auth | Réponse de la Gateway |
|---|---|
| `200` + `{ "user_id", "role" }` | requête proxifiée avec `X-User-Id` (+ `X-User-Role` si présent) |
| `200` mais JSON invalide / `user_id` vide | `500` |
| `401` | `401` (ou redirection login pour HTML) |
| `403` | `403` |
| autre code, timeout, erreur réseau | `503` |

Le corps de la réponse d'auth est limité à 64 Kio ; le client HTTP réutilise les connexions (keep-alive) pour tenir le budget de 100 ms.

## Codes de réponse propres à la Gateway

| Code | Cause |
|---|---|
| `401` | Token absent / mal formé, ou rejet par le service d'auth |
| `403` | Rejet par le service d'auth |
| `404` | Aucune route ne correspond |
| `413` | Corps > `max_body_bytes` |
| `429` | Quota de rate limiting dépassé |
| `500` | Réponse du service d'auth inexploitable |
| `502` | Backend injoignable / erreur de proxy |
| `503` | Service d'auth injoignable |
| `504` | Backend sans réponse après `timeout_seconds` |

## Observabilité

- **Logs JSON** sur stdout, une ligne par requête (`method`, `path`, `status`, `duration`, `bytes`, `ip`, `correlation_id`).
- **Correlation ID** : `X-Correlation-ID` réutilisé s'il est valide (≤ 128 caractères, alphanumériques + `.`, `_`, `-`), sinon UUID v4 généré ; propagé au service d'auth et aux backends.
- **Health check** : `GET /health` → `200 {"status":"ok"}`, répondu localement, exempté de rate limiting.

## Durcissement CORS en production

Le wildcard `allowed_origins: ["*"]` n'est toléré que si `environment` vaut explicitement `development`. En `production` (ou tout autre valeur), un wildcard **fait échouer le démarrage**. Poser `environment: production` dans `config.yaml` ou `GATEWAY_ENV=production`.

## Démarrage

```sh
go build -o bin/gateway ./cmd/gateway
./bin/gateway -config config.yaml
```

Arrêt propre sur `SIGINT` / `SIGTERM` (drain des connexions, 10 s de grâce).

> Note : la spécification OpenAPI n'est pas fournie pour la Gateway — elle ne définit pas de contrat propre mais relaie les APIs backend. Voir les `openapi.yaml` de chaque service pour les schémas de requêtes / réponses.
