# CH-Relay

**Broker MQTT 5.0** auto-hébergé, écrit en Rust. C'est le bus de messagerie de l'écosystème CustHome : les services publient et s'abonnent en **MQTT 5.0 standard**, avec n'importe quelle bibliothèque cliente (Rust, Go, Node, navigateur, mobile…). Il n'y a **pas de REST** : toute l'interaction passe par le protocole MQTT.

## Stack

- **Langage** : Rust, deux crates :
  - `relay-core` — moteur du broker (matching de topics, souscriptions, retained, sessions, machine QoS). Sans I/O, entièrement testable.
  - `relay-server` — daemon tokio : listeners TCP / WebSocket / TLS, codec MQTT 5 (`rmqtt-codec`), stockage sur disque (`redb`), redélivrance + dead-letter, journal d'événements, dashboard de supervision. Produit le binaire `relay`.

## Ports par défaut

| Transport | Adresse par défaut | Usage | Activation |
|---|---|---|---|
| **TCP** (`mqtt://`) | `127.0.0.1:1883` | backends (Rust, Go, Node…) | toujours |
| **WebSocket** (`ws://`) | `0.0.0.0:8083` | navigateurs et mobiles (sous-protocole `mqtt`) | toujours |
| **TLS** (`mqtts://`) | `0.0.0.0:8883` | connexions natives chiffrées | si `tls_cert` + `tls_key` |
| **Dashboard HTTP** | `http_addr` (ex. `127.0.0.1:8080`) | supervision (`/`, `/stats`) | si `http_addr` défini |

> Le listener WebSocket accepte l'upgrade sur n'importe quel chemin (`ws://host:8083` ou `ws://host:8083/mqtt`) et répond au sous-protocole `mqtt`.

## Configuration

Fichier TOML (chemin via la variable `RELAY_CONFIG`, défaut `config.toml`). Voir `config.toml.example` dans le repo. La section `[auth]` est **obligatoire** : le broker refuse de démarrer sans elle.

| Clé | Défaut | Description |
|---|---|---|
| `tcp_addr` | `127.0.0.1:1883` | Listener MQTT natif |
| `ws_addr` | `0.0.0.0:8083` | Listener MQTT-over-WebSocket |
| `data_dir` | — | Répertoire de persistance (`redb`) ; sans lui, tout est en mémoire |
| `max_delivery_attempts` | `5` | Tentatives avant dead-letter (1 = pas de retry) |
| `retry_base_secs` | `5` | Back-off de base (double à chaque tentative) |
| `retry_max_secs` | `60` | Plafond du back-off |
| `event_log_max` | `100000` | Taille max du journal d'événements (0 = désactivé) |
| `tls_addr` | `0.0.0.0:8883` | Listener TLS (si cert + key) |
| `tls_cert` / `tls_key` | — | Chemins PEM du certificat et de la clé |
| `http_addr` | — | Adresse du dashboard de supervision |
| `http_allow_external` | `false` | Autorise l'accès externe au dashboard |
| `max_connections` | `1000` | Connexions simultanées max |
| `connect_timeout_ms` | `5000` | Timeout de handshake CONNECT |
| `max_subscriptions_per_client` | `50` | Souscriptions max par client |

Certaines valeurs par défaut sont aussi surchargeables par variables d'environnement (`RELAY_HTTP_ALLOW_EXTERNAL`, `RELAY_MAX_CONNECTIONS`, `RELAY_CONNECT_TIMEOUT_MS`, `RELAY_MAX_SUBSCRIPTIONS_PER_CLIENT`, `RELAY_ALLOWED_AUDIENCES`).

## Authentification (obligatoire)

Chaque `CONNECT` doit porter un **JWT HS256 comme mot de passe MQTT** (le username est ignoré). Le broker vérifie :

- la **signature** (secret `jwt_secret` de `[auth]`, partagé avec l'Authenticator) ;
- l'**issuer** : `iss` doit valoir `ch-api-authenticator` ;
- l'**audience** : `aud` doit contenir au moins une valeur de `allowed_audiences` (défaut : `ch-api-drive`, `ch-api-budgy`, `ch-relay`) ;
- l'**expiration** (`exp` requis et non expiré).

Un token manquant, mal formé, de signature invalide, expiré, d'issuer ou d'audience non autorisés reçoit un `CONNACK` **Not authorized (0x87)** et la connexion est fermée.

### Paramètres `[auth]`

| Clé | Défaut | Description |
|---|---|---|
| `jwt_secret` | — (requis) | Secret HS256 de vérification |
| `identity_claim` | `sub` | Claim servant d'identité (et de `{sub}` dans l'ACL) |
| `roles_claim` | `roles` | Claim portant les rôles (tableau JSON) |
| `allowed_audiences` | `[ch-api-drive, ch-api-budgy, ch-relay]` | Audiences acceptées |
| `acl` | `[]` | Règles d'ACL par rôle |

## ACL par topic

L'ACL est construite en **unionnant** toutes les règles `[[auth.acl]]` dont le `role` correspond à un rôle du token (`role = "*"` correspond à tout client authentifié). Les motifs peuvent contenir des placeholders `{claim}` substitués depuis le token (ex. `drive/{sub}/#`). Un motif référençant un claim absent n'accorde rien (fail closed).

- **Publish** sur `T` : autorisé si un motif `publish` correspond au topic concret `T`.
- **Subscribe** avec filtre `F` : autorisé seulement si un motif `subscribe` **subsume** `F` (impossible d'élargir sa portée avec `#`).
- **Souscriptions partagées** (`$share/groupe/F`) : vérifiées sur le filtre interne `F`.
- **Replay** (`$replay/{from}/{filter}`) : vérifié sur l'ACL `subscribe` de `{filter}` (une lecture).

Un subscribe refusé renvoie `SUBACK Not authorized` pour ce filtre ; un publish refusé renvoie `PUBACK`/`PUBREC Not authorized` (QoS 1/2) ou est ignoré (QoS 0).

### Exemple d'ACL (config.toml.example)

| Rôle | publish | subscribe |
|---|---|---|
| `drive` | `drive/{sub}/#` | `drive/{sub}/#` |
| `drive_service` | `drive/+/files/+/uploaded` | (aucun) |
| `drive_admin` | `drive/#` | `drive/#`, `$dlq/#` |

## Fonctionnalités MQTT 5

| Besoin | Mécanisme MQTT 5.0 |
|---|---|
| File de travail (consommateurs concurrents) | Souscriptions partagées `$share/groupe/topic` |
| Diffusion pub/sub | Topics + wildcards `+` / `#` |
| Garanties de livraison | QoS 0 / 1 / 2 |
| Dernière valeur connue | Messages `retained` |
| Détection de service mort | Will (LWT) |
| TTL de message | Message Expiry Interval |
| Messages non livrables | Dead-letter queue `$dlq/#` + retry (extension Relay) |
| Rejeu / catch-up | Journal d'événements via `$replay/{from}/{filter}` (extension Relay) |

## Espaces de noms réservés

- `$share/{groupe}/{filtre}` — file de travail (round-robin, une livraison par message par groupe).
- `$dlq/{client}/{topic}` — messages QoS 1/2 non livrés (après `max_delivery_attempts` ou expiration de session durable) ; en lecture seule, produit par le broker.
- `$replay/{from_offset}/{filtre}` — requête de rejeu (publier en QoS 0) ; le broker renvoie les événements journalisés (`offset >= from_offset`) sur leurs topics d'origine, chacun avec la propriété utilisateur `x-replay-offset`.

Les filtres wildcard commençant par `+` ou `#` **ne matchent pas** les topics `$` ; s'abonner explicitement à `$dlq/#` pour voir les dead letters.

## Dashboard de supervision

Quand `http_addr` est défini :

- `GET /` — page HTML live (polling toutes les 2 s).
- `GET /stats` — snapshot JSON :

```json
{
  "clients_online": 3,
  "clients_total": 5,
  "subscriptions": 12,
  "retained": 4,
  "dead_letters": 0,
  "events": 1280,
  "next_offset": 1280
}
```

> Dans l'orchestration CustHome (`Tools/startup.sh`), le health check de Relay pointe sur `http://127.0.0.1:8084/stats` : le dashboard est donc exposé sur `8084` dans ce déploiement (via `http_addr`). L'exemple de `config.toml.example` utilise `8080` à titre indicatif.

## Démarrage

```sh
cargo run -p relay-server
```

## Conventions et intégration CustHome

- Les services CustHome se connectent en présentant leur JWT (émis par l'Authenticator) comme mot de passe MQTT.
- CH-Api-Drive publie les événements « fichier téléversé » (rôle `drive_service`, motif `drive/+/files/+/uploaded`).
- CH-Api-Authenticator publie l'événement de suppression d'utilisateur ; CH-Api-Budgy s'y abonne (topic `auth/user/deleted`) pour déclencher l'effacement des données.
- Le jeton de service consommé par Drive (`RELAY_SERVICE_TOKEN`) est régénéré par le script `Tools/rotate-relay-service-token.sh`.

> Note de cohérence : le `README.md` du repo présente l'authentification comme « opt-in » (héritage de la roadmap V3). La configuration actuelle (`config.toml.example`, `crates/relay-server/src/config.rs`) rend la section `[auth]` **obligatoire** — il n'y a plus de mode ouvert / anonyme.
