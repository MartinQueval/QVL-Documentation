# Écosystème CustHome — Back-end

CustHome est une plateforme multi-portail. Un **API Gateway** en Go sert de façade HTTP unique et route le trafic vers des microservices Rust (Axum) spécialisés, plus un **broker MQTT** pour la messagerie temps réel. L'authentification est centralisée et déléguée par la Gateway à un service dédié.

Cette documentation couvre uniquement le **back-end** (APIs, gateway, broker, outillage). Les portails front (`CH-Portal-*`, `CH-Portail-Admin`) sont documentés ailleurs.

## Architecture

```
Navigateur / portails (3200-320x)
              │
              ▼
   CH-Api-GateWay (Go)  :8180        façade HTTP unique
              │
   ┌──────────┼───────────────┬──────────────────┬──────────────────┐
   │          │               │                  │                  │
   ▼          ▼               ▼                  ▼                  ▼
/api/auth   /api/admin     /api/drive         /api/budgy         /api/maloe
/validate     │               │                  │                  │
   ▼          ▼               ▼                  ▼                  ▼
CH-Api-Authenticator     CH-Api-Drive       CH-Api-Budgy       (API Maloe)
   (Rust/Axum)  :8181     (Rust/Axum) :8182  (Rust/Axum) :8183      :8191
   MongoDB                PostgreSQL         PostgreSQL

CH-Relay (broker MQTT 5.0) :1883 (TCP) · :8083 (WebSocket) · :8883 (TLS)
   ▲ publient des événements métier : Authenticator, Drive, Budgy
```

## Ports par défaut

| Service | Rôle | Port | Stack | Stockage |
|---|---|---|---|---|
| CH-Api-GateWay | Façade HTTP, routage, auth déléguée, CORS, rate limit | 8180 | Go (stdlib) | — |
| CH-Api-Authenticator | Authentification multi-portail, comptes, rôles | 8181 | Rust / Axum | MongoDB |
| CH-Api-Drive | Fichiers et galerie média | 8182 | Rust / Axum | PostgreSQL + FS |
| CH-Api-Budgy | Budget personnel, agrégation bancaire | 8183 | Rust / Axum | PostgreSQL |
| API Maloe | 5e portail « Maloe » (audience `ml-api-maloe`, portail `portail_maloe`) | 8191 | — | — |
| CH-Relay | Broker MQTT 5.0 (bus d'événements) | 1883 / 8083 / 8883 | Rust | redb (opt.) |
| Missive | Envoi d'emails (dépendance externe du reset mot de passe) | 8184 | Rust | — |
| Portails front | CH-Portal-Authenticator / Admin / Drive / Budgy (+ Maloe) | 3200-320x | React | — |

## Repos documentés

| Repo | Description | Documentation |
|---|---|---|
| CH-Api-GateWay | API Gateway Go, routage par préfixe | [CH-Api-GateWay/README.md](CH-Api-GateWay/README.md) |
| CH-Api-Authenticator | Service d'authentification | [CH-Api-Authenticator/README.md](CH-Api-Authenticator/README.md) · [openapi.yaml](CH-Api-Authenticator/openapi.yaml) |
| CH-Api-Drive | Service de fichiers / galerie | [CH-Api-Drive/README.md](CH-Api-Drive/README.md) · [openapi.yaml](CH-Api-Drive/openapi.yaml) |
| CH-Api-Budgy | Service de budget bancaire | [CH-Api-Budgy/README.md](CH-Api-Budgy/README.md) · [openapi.yaml](CH-Api-Budgy/openapi.yaml) |
| CH-Relay | Broker MQTT 5.0 | [CH-Relay/README.md](CH-Relay/README.md) |
| Tools | Scripts d'orchestration | [Tools/README.md](Tools/README.md) |
| CH-Api-Maloe | IA auto-hébergée — **en conception, non implémenté** | [CH-Api-Maloe/CONCEPTION.md](CH-Api-Maloe/CONCEPTION.md) |

> 🗓️ **Décision (2026-08-21)** — ajout du document de conception de **Maloë**, IA auto-hébergée de
> CustHome (Ollama + tiers de modèles, identité versionnée, mémoire par utilisateur). Retenu en séance :
> Core en **Rust** (cohérence du dépôt), flux de génération via **CH-Relay** et non en SSE proxifié
> (le Gateway tue toute requête à 5 s), et accès aux API via un **grant explicite scopé et révocable**
> plutôt que par réutilisation des tokens utilisateur — le claim `ip` et le TTL de 15 min l'interdisent
> de fait. Non rattaché à un sprint.

## Flux d'authentification

L'authentification repose sur des **JWT HS256** émis par l'Authenticator et un modèle de **rôles par portail**.

1. Le front appelle `POST /api/auth/login` (route publique proxifiée). L'Authenticator vérifie les identifiants (Argon2id), émet un JWT (TTL 15 min) et pose deux cookies `HttpOnly` (`ch_token` pour l'access token, `ch_refresh` pour le refresh token).
2. Pour toute requête vers une route protégée, la Gateway appelle `GET /validate` sur l'Authenticator avec :
   - le token dans `Authorization: Bearer <token>` (ou via le cookie `ch_token`) ;
   - un header `X-Portal` indiquant le portail visé (ex. `portail_drive`), issu du champ `portal` de la route ;
   - un header `X-Client-IP` (IP client résolue). La Gateway le transmet toujours, mais il **ne sert plus au filtrage d'accès** (voir ci-dessous) ; il n'est consulté au `/validate` que pour d'anciens jetons portant encore le claim `ip`.
3. L'Authenticator valide le JWT **sans I/O** (signature, expiration), vérifie que le portail est connu et qu'un rôle est présent, puis renvoie `200 { "user_id", "role" }`.
4. La Gateway injecte alors `X-User-Id` (et `X-User-Role`) dans la requête proxifiée vers le backend. Ces headers entrants sont systématiquement supprimés en amont (anti-usurpation).

**Filtrage par appareil (`ch_device`)** — depuis le 2026-08-10, l'accès n'est plus restreint par whitelist d'adresses IP mais par **appareil reconnu**. Le code n'a plus ni champ `allowed_ips` ni claim JWT `ip`. Chaque utilisateur porte un drapeau `whitelist_only` et une liste `devices[]` ; un cookie `ch_device` (HttpOnly, non secret côté serveur — seul son hash est stocké) identifie le navigateur indépendamment du réseau. Tant que le compte n'est pas verrouillé, tout nouvel appareil s'inscrit de lui-même au login ; une fois `whitelist_only` actif, seuls les appareils reconnus peuvent se connecter (code `device_not_allowed`). Des garde-fous anti-verrouillage empêchent d'activer la restriction sans appareil reconnu ou de retirer le dernier appareil d'un compte restreint.

Le JWT porte les claims `sub`, `roles`, `iss` (`ch-api-authenticator`), `aud` (audience par portail : `ch-api-drive`, `ch-api-budgy`, `ml-api-maloe`), `iat`, `exp`. Le claim `ip` n'est **plus émis** (le champ subsiste uniquement pour valider d'anciens jetons). Le même `JWT_SECRET` est partagé entre l'Authenticator et les services qui valident les tokens localement (Drive, Budgy) ainsi que le broker Relay.

## Routage de la Gateway

La Gateway route par **préfixe de chemin** (`config.yaml`, section `routes`). Chaque route retire son préfixe (`strip_prefix: true`) avant de proxifier vers le backend.

| Préfixe | Destination | Auth requise | Portail (`X-Portal`) | Effet |
|---|---|---|---|---|
| `/api/auth` | `http://localhost:8181` | non | `portail_home` | Routes publiques (login, register, refresh…) |
| `/api/admin` | `http://localhost:8181` | oui | `portail_admin` | Administration des comptes et rôles |
| `/api/drive` | `http://localhost:8182` | oui | `portail_drive` | Service Drive (corps ≤ 17 Mo, timeout 120 s) |
| `/api/budgy` | `http://localhost:8183` | oui | `portail_budgy` | Service Budgy |
| `/api/maloe` | `http://localhost:8191` | oui | `portail_maloe` | 5e portail Maloe (`timeout_seconds: 0` → SSE, aucun timeout) |

Exemples : `POST /api/auth/login` → `POST /login` sur l'Authenticator ; `GET /api/drive/files` → `GET /files` sur Drive (Drive n'expose plus de préfixe `/v1`) ; `GET /api/budgy/v1/me` → `GET /v1/me` sur Budgy.

La validation des tokens des routes protégées passe par `auth_service_url` (`http://localhost:8181/validate`), appelée directement par la Gateway (hors table de routage).

## Conventions transverses

- **Versionnement d'API** : le préfixe `/v1` n'est **pas uniforme** entre services. **Budgy** et **Authenticator** exposent leurs routes applicatives sous `/v1` (l'Authenticator les monte aussi à la racine, sans préfixe, pour compatibilité). **Drive a abandonné `/v1` le 2026-07-22** (commit `bbe5b09`) : ses routes sont servies à la racine (`/files`, `/gallery`…), donc `GET /api/drive/files` via la Gateway. Les routes opérationnelles (`/health`, `/ping`) et internes (`/internal/...`) ne sont jamais versionnées.
- **Format d'erreur JSON** : les services renvoient un corps `{ "error": "...", "message": "..." }` (Authenticator, Drive) ou `{ "code": "...", "message": "..." }` (Budgy).
- **Contrat inter-services** : l'appel `POST /internal/users/resolve` de l'Authenticator (résolution nom/email) est protégé par le header `x-internal-secret` (secret `INTERNAL_API_SECRET` partagé).
- **Bus d'événements** : Authenticator, Drive et Budgy publient des événements métier (ex. suppression d'un utilisateur, fichier téléversé) sur CH-Relay en MQTT 5.0, avec authentification JWT.
