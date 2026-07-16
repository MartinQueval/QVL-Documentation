# Relay

## Rôle

Relay est un broker de messages **MQTT 5.0** auto-hébergé, écrit en Rust. C'est le bus de communication de la QVL-ToolBox : les services publient et souscrivent sur un protocole standard, si bien que n'importe quel client (Node/TypeScript, Go, Java, Rust, navigateur, mobile) se connecte avec une bibliothèque MQTT du commerce. Il n'y a **pas de REST et pas de SDK Relay** : on parle du MQTT 5 standard.

> Doublon connu : **CH-Relay** (organisation CustHome) est un doublon du même produit. La documentation de référence est celle-ci.

## Stack

- Rust, deux crates (même découpage que AIGate `*-core` / `*-server`) :
  - `relay-core` : moteur du broker (matching de topics, souscriptions, store retained, sessions, machine à états QoS), sans I/O, unitairement testable.
  - `relay-server` : démon Tokio (listeners TCP / WebSocket / TLS, codec de paquets MQTT via `rmqtt-codec`, store disque, timer de redélivrance et dead-letter, journal d'événements, dashboard de monitoring embarqué). Produit le binaire `relay`.

## Lancement et endpoints

```bash
cargo run -p relay-server
```

La configuration est lue depuis `config.toml` (voir `config.toml.example`), surchargée par la variable d'environnement `RELAY_CONFIG`.

| Transport | Adresse par défaut | Usage | Activé |
|---|---|---|---|
| TCP (`mqtt://`) | `127.0.0.1:1883` | backends (Rust, Go, Java, Node, Python) | toujours |
| WebSocket (`ws://`) | `0.0.0.0:8083` | navigateurs et mobiles (sous-protocole `mqtt`) | toujours |
| TLS (`mqtts://`) | `0.0.0.0:8883` | connexions natives chiffrées | quand `tls_cert` + `tls_key` sont définis |
| Dashboard HTTP | selon `http_addr` (ex. `127.0.0.1:8080`) | monitoring (`/`, `/stats`) | quand `http_addr` est défini |

Le listener WebSocket accepte l'upgrade sur n'importe quel chemin et répond au sous-protocole `mqtt`.

## Topics et matching

- Un **topic name** (publication) a des niveaux séparés par `/` (`orders/eu/created`), sans wildcard.
- Un **topic filter** (souscription) peut utiliser des wildcards :
  - `+` : exactement un niveau (`orders/+/created`).
  - `#` : le reste de l'arbre, en dernière position (`orders/#`).
- Les **topics `$` sont protégés** : un filtre commençant par `+` ou `#` ne matche pas les topics débutant par `$` (par exemple `$dlq/...`). Il faut les souscrire explicitement (`$dlq/#`).

Le matching de filtres vit dans `crates/relay-core/src/router.rs`. Le hub qui orchestre connexions, sessions et fan-out vit dans `crates/relay-server/src/hub.rs`.

Namespaces réservés :

| Namespace | Sens |
|---|---|
| `$share/{group}/{filter}` | Souscriptions partagées : file de consommateurs concurrents (round-robin, une livraison par message et par groupe). |
| `$dlq/{client}/{topic}` | Dead-letter queue en lecture seule, produite par le broker pour les messages non livrés/expirés. |
| `$replay/{from}/{filter}` | Contrôle de replay : publier (QoS 0, payload vide) rejoue les événements journalisés d'offset ≥ `from` matchant `{filter}`. |

## Fonctionnalités MQTT

- QoS 0 / 1 / 2 (le broker accorde jusqu'à QoS 2 ; QoS effectif = min(QoS publication, QoS accordé)).
- Messages retained (dernière valeur d'un topic, rejouée aux souscripteurs tardifs ; payload vide + retain efface).
- Will (Last Will & Testament) publié à la déconnexion anormale.
- Sessions durables (`clean_start=false` + expiry > 0) : souscriptions et messages QoS 1/2 non acquittés conservés et retransmis à la reconnexion.
- Souscriptions partagées `$share` (work queue à consommateurs concurrents).
- Dead-letter queue avec retry à back-off exponentiel (`retry_base_secs`, doublement, plafonné à `retry_max_secs`, dead-letter après `max_delivery_attempts`).
- Replay/event-sourcing depuis un offset via `$replay/{from}/{filter}`.
- TLS (mqtts) via rustls, activé en pointant `tls_cert` + `tls_key` sur des fichiers PEM.
- Persistance disque (store embarqué `redb`, opt-in via `data_dir`) : retained, sessions durables, files in-flight, dead letters et journal d'événements survivent au redémarrage.

## Configuration (config.toml)

Toutes les clés sont optionnelles ; valeurs par défaut indiquées.

```toml
tcp_addr = "127.0.0.1:1883"
ws_addr  = "0.0.0.0:8083"

# Persistance (désactivée par défaut). Fichier redb relay.redb créé dans ce dossier.
# data_dir = "/var/lib/relay"

# Redélivrance / dead-letter
max_delivery_attempts = 5
retry_base_secs       = 5
retry_max_secs        = 60

# Journal d'événements / replay (nécessite data_dir). 0 désactive.
event_log_max = 100000

# TLS (mqtts). Activé quand cert et key sont tous deux définis.
# tls_addr = "0.0.0.0:8883"
# tls_cert = "/etc/relay/cert.pem"
# tls_key  = "/etc/relay/key.pem"

# Dashboard de monitoring embarqué (désactivé par défaut).
# http_addr = "127.0.0.1:8080"
```

## Authentification et ACL

L'authentification est **opt-in et générique** (implémentée dans `crates/relay-server/src/auth.rs`). Sans bloc `[auth]`, le broker est ouvert (mode legacy). Avec un bloc `[auth]`, chaque CONNECT doit porter un **JWT HS256 comme mot de passe MQTT** (le username est ignoré) ; un jeton absent, malformé, mal signé ou expiré reçoit un CONNACK `Not authorized (0x87)`.

Un ACL de topics par rôle s'applique alors, templaté avec les claims du jeton :

- Publier sur `T` est autorisé si un motif `publish` matche `T`.
- Souscrire au filtre `F` n'est autorisé que si un motif `subscribe` *subsume* `F` (pas d'élargissement avec `#`).
- Les souscriptions partagées `$share/g/F` sont vérifiées sur le filtre interne `F`.
- Une requête `$replay/{from}/{filter}` est vérifiée sur l'ACL de souscription pour `{filter}`.
- Un motif référençant un claim absent du jeton n'accorde rien (fail closed).

## Dashboard de monitoring

Quand `http_addr` est défini (dashboard embarqué dans `crates/relay-server/src/dashboard.rs`) :

- `GET /` : page HTML live qui interroge les stats toutes les 2 s.
- `GET /stats` : instantané JSON.

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

## Guide d'intégration

Le guide complet (connexion par langage, concepts MQTT tels que Relay les applique, namespaces réservés, patterns et aide-mémoire pour agents IA) vit dans le fichier `docs/USAGE.md` du dépôt source Relay.
