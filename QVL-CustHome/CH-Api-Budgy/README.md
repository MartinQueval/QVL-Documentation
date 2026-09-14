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
- **Version** : v1.0.2 (tag Git ; `Cargo.toml` porte encore `0.1.0`)

## Port par défaut

`8183` (clé `server.port` de `config.toml`, surchargée par la variable `PORT`).

## Concepts clés

- **Propriétaire (`ProprietaireId`)** : identifiant du compte (le `sub` du JWT). Toutes les données sont isolées par propriétaire.
- **Consentement (`consent`)** : autorisation d'accès aux comptes d'un établissement. Cycle de vie : `pending` → `active`, sinon `expired`, `revoked`, `failed`. Flux : `POST /consents` (initie, renvoie une URL d'autorisation) → redirection banque → `POST /consents/callback` (finalise, enregistre les comptes) → renouvellement via `POST /consents/{id}/renew`.
- **État de renouvellement** : chaque consentement expose `renewal` (`up-to-date`, `renewal-required`, `expired`) et `renewable`, calculés par rapport à une marge.
- **Comptes bancaires** : IBAN masqué, devise, solde optionnel (montant en centimes, type `available`/`booked`/`expected`).
- **Transactions** : libellé, montant en centimes, devise, statut (`booked`/`pending`), dates de comptabilisation et de valeur. Chaque transaction expose aussi son **`clean_label`** : le tiers extrait du libellé bancaire (marchand ou contrepartie), débarrassé des préfixes d'opération, dates, masques de carte et références. C'est cette forme stable — et non le libellé brut, qui change chaque mois — qui sert à l'affichage, au matching des règles et au regroupement des récurrences.
- **Catégorisation** : `categorization_source` vaut `manual`, `rule` ou `none`. Les traitements automatiques ne touchent **que** les transactions en `none` : un choix manuel n'est jamais réécrit.
- **Virements internes** : un mouvement entre deux comptes du même propriétaire, apparié automatiquement. Ni dépense ni revenu — sans quoi il serait compté deux fois — donc exclu de tous les agrégats. La transaction expose alors `is_internal_transfer: true` (affiché « Virement interne, non compté » côté portail). L'appariement est décrit plus bas (v1.0.2).
- **Enveloppes** : « budgets » libres et manuels, indépendants d'un mois ou d'une catégorie (suivre un projet : vacances, achat…). L'utilisateur y affecte manuellement des transactions ; aucune règle n'affecte automatiquement une enveloppe. Une enveloppe porte un nom (≤ 30 caractères), une icône, une couleur et un `montant_cents` cible, et l'API renvoie la consommation calculée (`depense_cents`, `restant_cents`, `pourcentage_consomme`, `depasse`, `nombre_transactions`). À distinguer des **budgets** mensuels par catégorie.
- **Préférences — jour de départ du mois** : le cycle budgétaire est réglable via `jour_debut_mois` (1 à 31, défaut `1`). Un jour > 1 décale le cycle sur le mois calendaire précédent (ex. jour 28 → le cycle « août » court du 28 juillet au 27 août) ; un jour comme 31 est ramené au dernier jour existant du mois. Un cycle porte le nom du mois de son **dernier** jour.
- **Worker de synchronisation** (optionnel) : rafraîchit périodiquement comptes et transactions dans une fenêtre glissante, avec quota journalier. Le premier cycle part immédiatement au démarrage du service.

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
| `worker_synchro.fenetre_transactions_jours` | `30` | Fenêtre de récupération des transactions. **`88` en production** : Enable Banking refuse toute demande à 90 jours ou plus (`422 WRONG_TRANSACTIONS_PERIOD`) et la synchronisation échoue alors sans rien insérer. |

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
| PUT | `/accounts/{account_id}/transactions/{transaction_id}/category` | JWT budgy | Catégorise manuellement une transaction | 200, 400, 401, 403, 404 |
| POST | `/accounts/{account_id}/transactions/{transaction_id}/rule` | JWT budgy | Crée une règle **dérivée du libellé** de la transaction, puis l'applique rétroactivement | 201, 400, 401, 403, 404 |
| GET | `/transactions` | JWT budgy | Transactions du propriétaire tous comptes confondus (filtres `account_id`, `category_id`, `from`/`to`, `type` ; tri `date`/`amount`) | 200, 400, 401, 403 |
| POST | `/transactions/recategoriser` | JWT budgy | Réconciliation idempotente : virements internes → règles → crédits | 200, 401, 403 |
| PUT | `/transactions/{transaction_id}/enveloppe` | JWT budgy | Affecte/retire une transaction d'une enveloppe (corps `{ enveloppe_id }`, `null` = retirer) | 204, 400, 401, 403, 404 |
| GET | `/balance` | JWT budgy | Solde consolidé tous comptes, avec solde à venir si la banque l'expose | 200, 401, 403 |
| GET | `/categories` | JWT budgy | Catégories système et propres au propriétaire | 200, 401, 403 |
| POST | `/categories` | JWT budgy | Crée une catégorie | 201, 400, 401, 403, 409 |
| PUT | `/categories/{category_id}` | JWT budgy | Modifie une catégorie du propriétaire | 200, 400, 401, 403, 404 |
| DELETE | `/categories/{category_id}` | JWT budgy | Supprime une catégorie du propriétaire | 204, 401, 403, 404 |
| POST | `/categorization-rules` | JWT budgy | Crée une règle de catégorisation (motif saisi) | 201, 400, 401, 403, 404 |
| GET | `/budgets` | JWT budgy | Budgets mensuels par catégorie (`mois=YYYY-MM`) | 200, 400, 401, 403 |
| POST | `/budgets` | JWT budgy | Crée ou met à jour le budget d'une catégorie pour un mois | 201, 400, 401, 403, 404 |
| GET | `/budgets/remaining` | JWT budgy | Reste à dépenser par catégorie (`month=YYYY-MM`), budgété ou prédit | 200, 400, 401, 403 |
| GET | `/preferences` | JWT budgy | Préférences du propriétaire (dont `jour_debut_mois`) | 200, 401, 403 |
| PUT | `/preferences` | JWT budgy | Met à jour les préférences (`jour_debut_mois` : 1 à 31) | 200, 400, 401, 403 |
| GET | `/enveloppes` | JWT budgy | Liste des enveloppes avec consommation calculée | 200, 401, 403 |
| POST | `/enveloppes` | JWT budgy | Crée une enveloppe | 201, 400, 401, 403 |
| PUT | `/enveloppes/{enveloppe_id}` | JWT budgy | Modifie une enveloppe | 200, 400, 401, 403, 404 |
| DELETE | `/enveloppes/{enveloppe_id}` | JWT budgy | Supprime une enveloppe | 204, 401, 403, 404 |
| GET | `/expenses/by-category` | JWT budgy | Dépenses du mois réparties par catégorie | 200, 400, 401, 403 |
| GET | `/forecast` | JWT budgy | Budget prévisionnel du mois | 200, 400, 401, 403 |
| GET | `/banks` | JWT budgy | Liste des établissements bancaires disponibles | 200, 401, 403 |
| GET | `/consents` | JWT budgy | Liste des consentements du propriétaire, dédupliqués par établissement | 200, 401, 403 |
| POST | `/consents` | JWT budgy | Initie un consentement → URL d'autorisation | 200, 400, 401, 403, 502 |
| POST | `/consents/callback` | JWT budgy | Finalise le consentement (code + state) | 200, 400, 401, 403, 404, 409, 502 |
| POST | `/consents/{consent_id}/renew` | JWT budgy | Renouvelle un consentement éligible | 200, 401, 403, 404, 409, 502 |

### Opérationnel

| Méthode | Chemin | Auth | Description | Réponses |
|---|---|---|---|---|
| GET | `/health` | non | État du service | 200 |

## Calculs et prédictions

Trois mécanismes portent l'intelligence du service. Ils partagent une contrainte : **les libellés et les montants sont chiffrés en base**, donc aucun regroupement ni filtrage monétaire n'est possible en SQL — tout se fait en applicatif après déchiffrement.

### Catégorisation automatique

Une règle associe un motif de libellé à une catégorie, avec une priorité. Le matching compare les **tiers extraits** de part et d'autre, jamais les libellés bruts : un motif dérivé d'un libellé nettoyé (« CARTE INTERMARCHE ») ne serait sinon jamais reconnu dans « CARTE 07/07/26 INTERMARCHE CB*7513 », où la date s'intercale — et le format diffère d'une banque à l'autre pour le même marchand. Corollaire : un motif réduit à un préfixe d'opération (« ACHAT », « CARTE ») ne matche rien, ces préfixes étant retirés des deux côtés.

Les règles s'appliquent à l'insertion de chaque transaction et rétroactivement (création de règle, ou réconciliation). Les crédits encore non catégorisés basculent en « Salaire ».

### Prédiction par médiane

En l'absence de budget défini, le reste à dépenser et les revenus prévisionnels se prédisent sur la **médiane des 3 derniers mois**, catégorie par catégorie (un mois sans montant compte pour zéro). La médiane — et non la moyenne ni le seul mois précédent — évite qu'une dépense exceptionnelle isolée ne devienne une enveloppe mensuelle.

Le `total` du reste à dépenser est **exactement la somme des lignes renvoyées** : y agréger les dépenses non catégorisées produirait un total que le détail ne permet pas de recouper.

### Revenus récurrents

Les revenus du prévisionnel viennent de la **médiane des crédits mensuels par catégorie de revenu**, et non de la détection de récurrence. Celle-ci exige un montant fixe (±1 €) et un tiers identique : un salaire, qui varie de plusieurs dizaines d'euros et porte le mois dans son libellé, ne peut structurellement pas être reconnu. Un crédit rangé dans une catégorie de dépense est un remboursement et ne compte pas comme revenu. Les crédits sont exclus du calcul par récurrence, sous peine d'être comptés deux fois.

Les **dépenses** récurrentes, elles, restent détectées par occurrences à montant fixe (≥ 3 occurrences, intervalles de 26 à 35 jours) : le modèle convient aux abonnements et charges.

### Appariement des virements internes (v1.0.2)

L'ancienne heuristique — « même montant, comptes différents, appariés au premier crédit venu dans une fenêtre de ±4 jours » — appariait parfois la mauvaise face : deux débits de 200 € face à un seul crédit de 200 € sortaient à tort une épargne des totaux ; un virement de 10 € vers un tiers se mariait à un bonus de 10 € le lendemain, effaçant silencieusement à la fois une dépense et un revenu.

La logique actuelle **classe les couples candidats par ressemblance de libellé** :

1. On forme **tous** les couples débit/crédit plausibles : comptes différents, montants exactement opposés, écart de dates ≤ **4 jours** (`TOLERANCE_JOURS`).
2. Chaque couple est noté par une **similarité de Jaccard** sur les mots des libellés (intersection / union, en majuscules).
3. Les couples sont triés par similarité décroissante (avec départages déterministes par date puis identifiants, pour un résultat indépendant de l'ordre de lecture en base), puis acceptés gloutonnement — une face déjà consommée est ignorée.
4. Un couple dont le score est inférieur à `RESSEMBLANCE_MINIMALE` (= **0.25**) est **refusé**, sauf s'il est le seul appariement possible des deux côtés et qu'aucune des deux transactions n'a été rangée à la main (garde-fou anti-coïncidence de montant ambiguë ; la catégorisation manuelle est un indice, pas un veto).

Les transactions ainsi appariées portent `is_internal_transfer: true` et sont exclues de tous les agrégats.

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
