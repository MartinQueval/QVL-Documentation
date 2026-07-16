# Tools

Outillage et scripts d'orchestration transverses de l'écosystème CustHome. Ce repo se clone à la racine du projet, au même niveau que les repos de services.

```
CustHome/
├── CH-Api-GateWay/
├── CH-Api-Authenticator/
├── CH-Api-Drive/
├── CH-Api-Budgy/
├── CH-Relay/
├── CH-Portal-Authenticator/  CH-Portail-Admin/  CH-Portal-Drive/  CH-Portal-Budgy/
├── Tools/              ← ce repo
│   ├── startup.sh
│   ├── shutdown.sh
│   ├── audit.sh
│   ├── rotate-relay-service-token.sh
│   ├── MasterEnv/
│   └── HealthServ/
└── .masterenv
```

## startup.sh

Lance tous les services CustHome dans le bon ordre (Windows Git Bash et Linux).

```bash
./startup.sh                 # mode production
./startup.sh --dev           # mode développement (Vite HMR, cargo run sans --release)
./startup.sh --canop-local   # lie CanopUI en local (npm link) avant les portails
./startup.sh --canop-unlink  # délie CanopUI, réinstalle le package hébergé, puis quitte
./startup.sh --help          # aide
```

### Séquence de démarrage

1. **MasterEnv** — synchronise les variables d'environnement entre tous les repos (bloquant).
2. **Bases de données** — vérifie les BDD critiques (arrêt si échec) puis non-critiques (warning).
3. **Services critiques** — si un seul échoue, tout s'arrête.
4. **Services non-critiques** — warning si indisponible, on continue.

### Services orchestrés et health checks

| Catégorie | Service | Répertoire | Health check |
|---|---|---|---|
| Critique | CH-Api-GateWay | `CH-Api-GateWay` | `http://localhost:8180/health` |
| Critique | CH-Api-Authenticator | `CH-Api-Authenticator` | `http://localhost:8181/health` |
| Critique | CH-Relay | `CH-Relay` | `http://127.0.0.1:8084/stats` |
| Critique | CH-Portal-Authenticator | `CH-Portal-Authenticator` | `http://localhost:3200/health` |
| Non-critique | CH-Portail-Admin | `CH-Portail-Admin` | `http://localhost:3201/health` |
| Non-critique | CH-Api-Drive | `CH-Api-Drive` | `http://localhost:8182/health` |
| Non-critique | CH-Api-Budgy | `CH-Api-Budgy` | `http://localhost:8183/health` |
| Non-critique | CH-Portal-Drive | `CH-Portal-Drive` | `http://localhost:3202/health` |
| Non-critique | CH-Portal-Budgy | `CH-Portal-Budgy` | `http://localhost:3203/health` |
| Non-critique | Missive | `$MISSIVE_DIR` (hors arbre) | `http://localhost:8184/health` |

Bases vérifiées : MongoDB Auth (`localhost:27017`, critique) et PostgreSQL Drive (`localhost:5432`, non-critique).

### Bases de données et services

Le format de déclaration dans le script :

```bash
# Services : "nom|répertoire|commande_dev|commande_prod|url_health|timeout_sec"
# Bases    : "type|label|host|port|uri"   (types : mongo, postgres, mysql, redis)
```

### Librairie UI (canopui)

Les 4 portails consomment le design system via le package npm hébergé **`canopui`** (registry privé). `startup.sh` ne builde rien côté lib UI en fonctionnement normal ; `npm install` suffit. Les options `--canop-local` / `--canop-unlink` servent uniquement au développement de la librairie (chemin surchargeable via `CANOP_DIR`).

## shutdown.sh

Arrête proprement tous les services lancés par `startup.sh`, dans l'ordre inverse du démarrage.

```bash
./shutdown.sh          # arrêt gracieux (SIGTERM, 10 s max par service)
./shutdown.sh --force  # kill immédiat
```

Chaque service est identifié par son port d'écoute : Portal-Budgy 3203, Portal-Drive 3202, Api-Budgy 8183, Api-Drive 8182, Portail-Admin 3201, Portal-Authenticator 3200, Missive 8184, CH-Relay 1883, Api-Authenticator 8181, Gateway 8180.

## audit.sh

Audit de sécurité des dépendances Rust via `cargo audit` sur les repos Rust (CH-Api-Authenticator, CH-Api-Drive, CH-Api-Budgy, CH-Relay). **Bloquant** : sort avec un code non-zéro dès qu'une vulnérabilité est remontée.

```bash
./audit.sh                              # audit de tous les repos Rust
./audit.sh --ignore RUSTSEC-XXXX-YYYY   # ignorer une advisory (répétable)
```

Une advisory peut aussi être ignorée durablement via un fichier `.cargo/audit.toml` committé dans le repo concerné.

## rotate-relay-service-token.sh

Régénère le JWT de service `RELAY_SERVICE_TOKEN` (HS256) consommé par CH-Api-Drive comme mot de passe MQTT au CONNECT vers CH-Relay. Le token est signé avec le `jwt_secret` de CH-Relay (`config.toml`, section `[auth]`), source de vérité lue à l'exécution.

Claims posés : `sub=svc-drive`, `roles=["drive_service"]`, `aud=ch-relay`, `iss=ch-api-authenticator`, `iat`, `exp` (TTL par défaut 30 jours).

```bash
./rotate-relay-service-token.sh            # génère et affiche le token
./rotate-relay-service-token.sh --write    # génère ET pose dans .masterenv (sauvegarde .masterenv.bak)
./rotate-relay-service-token.sh --ttl 30   # TTL en jours (défaut 30)
```

Prérequis : `node` (≥ 18). Après `--write`, redémarrer CH-Api-Drive pour recharger `.masterenv` (le token est lu une fois au boot).

## MasterEnv

Gestionnaire centralisé des variables d'environnement (Rust). Scanne tous les repos, détecte les conflits de variables et de ports, et synchronise depuis un fichier `.masterenv` unique.

```bash
cd MasterEnv
cargo run --release -- --path ../..          # sync silencieux (GUI si conflits)
cargo run --release -- --path ../.. --edit   # éditeur GUI
cargo run --release -- --path ../.. --check  # rapport des ports (stdout)
```

## HealthServ

Service de maintenance système (redémarrage programmé, nettoyage RAM, auto-lancement de services), en Rust. En cours de refonte.
