# CH-Api-Budgy

Microservice **Budgy** de l'écosystème CustHome : budget personnel avec agrégation bancaire. Il gère le rattachement d'établissements bancaires (via consentement), l'exposition des comptes et de leurs soldes, et la lecture des transactions.

Il est exposé par [CH-Api-GateWay](../CH-Api-GateWay/README.md) sous le préfixe `/api/budgy` (auth requise, portail `portail_budgy`).

## Stack

- **Langage** : Rust (edition 2024), architecture hexagonale (domaine / ports / adapters)
- **Framework HTTP** : Axum 0.8
- **Base de données** : PostgreSQL (`sqlx`)
- **Chiffrement** : ChaCha20-Poly1305 (données sensibles) ; clé fournie par l'environnement
- **Source bancaire** : adaptateur `mock` ou `enablebanking` (API Enable Banking)
- **Auth** : JWT HS256 validé localement, rôle `budgy` requis
- **Bus** : abonnement MQTT (Relay) aux événements (ex. suppression d'utilisateur → effacement) et worker de synchronisation
- **Version** : 0.1.0

## Port par défaut

`8183` (clé `server.port` de `config.toml`, surchargée par la variable `PORT`).

## Concepts clés

- **Propriétaire (`ProprietaireId`)** : identifiant du compte (le `sub` du JWT). Toutes les données sont isolées par propriétaire.
- **Consentement (`consent`)** : autorisation d'accès aux comptes d'un établissement. Cycle de vie : `pending` → `active`, sinon `expired`, `revoked`, `failed`. Flux : `POST /consents` (initie, renvoie une URL d'autorisation) → redirection banque → `POST /consents/callback` (finalise, enregistre les comptes) → renouvellement via `POST /consents/{id}/renew`.
- **État de renouvellement** : chaque consentement expose `renewal` (`up-to-date`, `renewal-required`, `expired`) et `renewable`, calculés par rapport à une marge.
- **Comptes bancaires** : IBAN masqué, devise, solde optionnel (montant en centimes, type `available`/`booked`/`expected`).
- **Transactions** : libellé, montant en centimes, devise, statut (`booked`/`pending`), dates de comptabilisation et de valeur.
- **Worker de synchronisation** (optionnel) : rafraîchit périodiquement comptes et transactions dans une fenêtre glissante, avec quota journalier.

## Configuration

Configuration non sensible dans `config.toml`, surchargeable par variables `CH__` et par des variables dédiées. Secrets par environnement.

### config.toml

| Section / clé | Défaut | Description |
|---|---|---|
| `server.port` | `8183` | Port d'écoute |
| `server.log_level` | `INFO` | Verbosité |
| `token.issuer` | `ch-api-authenticator` | Issuer JWT attendu |
| `token.audience` | `ch-api-budgy` | Audience JWT attendue |
| `bank.source` | `mock` | Source bancaire (`mock` ou `enablebanking`) |
| `bank.callback_url` | `https://budgy.custhome.app/banque/callback` | URL de retour du consentement |
| `bank.enable_banking.base_url` | `https://api.enablebanking.com` | Base de l'API Enable Banking |
| `relay.enabled` | `false` | Abonnement au bus Relay |
| `relay.url` | `mqtt://127.0.0.1:1883` | Broker Relay |
| `relay.topic_user_deleted` | `auth/user/deleted` | Topic écouté pour l'effacement |
| `relay.topic_prefix` | `budgy` | Préfixe des topics publiés |
| `worker_synchro.enabled` | `false` | Worker de synchronisation |
| `worker_synchro.interval_secondes` | `21600` | Intervalle de synchronisation |
| `worker_synchro.quota_journalier` | `4` | Synchronisations max par jour et par compte |
| `worker_synchro.fenetre_transactions_jours` | `30` | Fenêtre de récupération des transactions |

### Variables d'environnement

| Variable | Requis | Description |
|---|---|---|
| `DATABASE_URL` | oui | Connexion PostgreSQL |
| `BUDGY_ENCRYPTION_KEY` | oui | Clé de chiffrement (base64, 32 octets) |
| `JWT_SECRET` | oui | Secret HS256 (≥ 32 octets, identique à l'Authenticator) |
| `PORT` | non | Surcharge `server.port` |
| `JWT_ISSUER`, `JWT_AUDIENCE` | non | Contrat JWT |
| `BANK_SOURCE`, `BANK_CALLBACK_URL` | non | Surcharge de la source bancaire |
| `ENABLE_BANKING_APP_ID`, `ENABLE_BANKING_PRIVATE_KEY_PEM`, `ENABLE_BANKING_PRIVATE_KEY_PATH`, `ENABLE_BANKING_REDIRECT_URL`, `ENABLE_BANKING_BASE_URL` | si `enablebanking` | Identifiants Enable Banking |
| `RELAY_ENABLED`, `RELAY_URL`, `RELAY_CLIENT_ID`, `RELAY_TOPIC_PREFIX`, `RELAY_EVENT_ISSUER`, `RELAY_SERVICE_TOKEN`, `RELAY_JWT_PRIVATE_KEY` | si `relay.enabled` | Configuration du bus Relay |
| `WORKER_SYNCHRO_ENABLED`, `WORKER_SYNCHRO_INTERVAL_SECONDES`, `WORKER_SYNCHRO_QUOTA_JOURNALIER`, `WORKER_SYNCHRO_FENETRE_JOURS` | non | Configuration du worker |

## Endpoints

Toutes les routes applicatives sont sous le préfixe **`/v1`** et exigent un JWT Bearer portant le rôle `budgy`. Via la Gateway : `/api/budgy/v1/...`.

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/me` | JWT budgy | Identité et rôles du propriétaire courant | 200, 401, 403 |
| GET | `/accounts` | JWT budgy | Liste paginée des comptes avec solde | 200, 400, 401, 403 |
| GET | `/accounts/{account_id}` | JWT budgy | Détail d'un compte avec solde | 200, 401, 403, 404 |
| GET | `/accounts/{account_id}/transactions` | JWT budgy | Transactions paginées d'un compte | 200, 400, 401, 403, 404 |
| GET | `/banks` | JWT budgy | Liste des établissements bancaires disponibles | 200, 401, 403 |
| GET | `/consents` | JWT budgy | Liste des consentements du propriétaire | 200, 401, 403 |
| POST | `/consents` | JWT budgy | Initie un consentement → URL d'autorisation | 200, 400, 401, 403, 502 |
| POST | `/consents/callback` | JWT budgy | Finalise le consentement (code + state) | 200, 400, 401, 403, 404, 409, 502 |
| POST | `/consents/{consent_id}/renew` | JWT budgy | Renouvelle un consentement éligible | 200, 401, 403, 404, 409, 502 |

### Opérationnel

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/health` | non | État du service | 200 |

## Pagination

Query params `limit` (défaut 50, max 200) et `offset` (défaut 0). `limit = 0` ou `limit > 200` renvoie `400 bad_request`.

### Enveloppe de liste

```json
{ "data": [ ... ], "total": 1234 }
```

`total` est le nombre total d'éléments correspondant au filtre, indépendamment de la pagination.

## Format d'erreur

```json
{ "code": "bad_request", "message": "limit ne peut pas dépasser 200" }
```

| `code` | Statut | Cas |
|---|---|---|
| `bad_request` | 400 | Validation (pagination, bank_id, state…) |
| `unauthorized` | 401 | Token invalide ou absent |
| `forbidden` | 403 | Rôle `budgy` manquant |
| `not_found` | 404 | Compte ou consentement introuvable |
| `conflict` | 409 | Consentement non éligible au renouvellement |
| `consentement_refuse` | 409 | Consentement refusé par la banque |
| `banque_indisponible` | 502 | Source bancaire injoignable |
| `internal_error` | 500 | Erreur interne |

## Montants

Les montants monétaires sont exprimés en **centimes** (entier signé, champ `amount_cents`).

## Spécification OpenAPI

Voir [openapi.yaml](openapi.yaml).
