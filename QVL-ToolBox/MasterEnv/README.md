# MasterEnv

## Rôle

MasterEnv est un gestionnaire centralisé et multi-plateforme des variables d'environnement pour les projets multi-repos. Il scanne les dépôts d'un projet, rassemble toutes les variables de configuration dans un fichier central `.masterenv`, détecte les conflits et les collisions de ports, et fournit une interface graphique pour les résoudre. Il fonctionne par **synchronisation**, pas par injection : il ne crée jamais une variable dans un fichier où la clé n'existe pas déjà.

## Stack

- Rust, CLI via `clap`, GUI native (OpenGL/WGPU).
- Support multi-plateforme : Windows, Linux, macOS.

## Ligne de commande

Le binaire scanne par défaut le répertoire parent (racine du projet) et ignore son propre dossier.

| Invocation | Effet |
|---|---|
| `masterenv` | Scan headless : construit/met à jour `.masterenv` et synchronise. Ouvre la GUI automatiquement si des conflits ou des ports occupés sont détectés. |
| `masterenv --edit` | Ouvre l'éditeur graphique complet quel que soit l'état. |
| `masterenv --check` | Affiche sur stdout le rapport des ports (libre/occupé), sans GUI ni écriture. |
| `masterenv --path <chemin>` | Cible une autre racine de projet que le parent du dossier MasterEnv. |

## Configuration (app-config.toml)

```toml
config_files = [".env", ".toml", ".ini", ".conf", ".cfg", ".properties", ".yaml", ".yml"]
ignored_directories = ["target", ".git", "node_modules", ".idea", ".vscode", "dist", "build", "__pycache__"]
```

- `config_files` : extensions de fichiers scannés pour des paires `KEY=VALUE`.
- `ignored_directories` : répertoires exclus du scan récursif.

## Fichier `.masterenv`

Source de vérité centrale. Les variables sont groupées par dépôt, séparées par des en-têtes `####### repo-name #######` :

```properties
####### api-admin #######
ADMIN_PORT=3000
ADMIN_DB_HOST=localhost

####### front-admin #######
FRONT_PORT=3001
ADMIN_API_URL=http://localhost:3000
```

Une variable ne peut pas apparaître dans deux sections. Si la même clé existe dans plusieurs repos avec des valeurs différentes, MasterEnv la détecte comme un **conflit**.

## Détection de conflits et règle d'ownership

Quand une même variable existe dans plusieurs repos avec des valeurs différentes, tous les conflits sont rassemblés dans un écran de résolution unique. La règle d'ownership : la variable appartient au repo qui la « crée » ; les autres repos qui l'utilisent doivent en recevoir la valeur. Chaque conflit se résout en choisissant la valeur d'un repo ou en saisissant une valeur personnalisée, puis via **Save & Sync**.

## Gestion des ports

MasterEnv repère automatiquement les variables dont le nom contient `port` (insensible à la casse).

- **Ports spécifiques** : pour chaque variable de port, tentative de bind sur `127.0.0.1:<port>` pour vérifier la disponibilité.
- **Ports système** : liste des ports occupés via les outils natifs de l'OS.

Un port occupé est mis en évidence dans la GUI, avec un champ de saisie d'un nouveau port validé en temps réel ; il est impossible d'enregistrer un port occupé.

## Synchronisation

À l'enregistrement, MasterEnv :

1. écrit `.masterenv` avec toutes les valeurs résolues ;
2. scanne les fichiers de configuration de chaque repo ;
3. pour chaque ligne `KEY=VALUE` où `KEY` existe dans `.masterenv`, remplace la valeur par la valeur maîtresse ;
4. préserve le formatage des fichiers ;
5. n'écrit que les fichiers réellement modifiés.

## Support multi-plateforme

| OS | Détection des ports | GUI |
|---|---|---|
| Windows | `netstat -ano -p TCP` + `TcpListener::bind` | native (OpenGL/WGPU) |
| Linux | `ss -tlnp` + `TcpListener::bind` | native (OpenGL/WGPU) |
| macOS | `lsof -iTCP -sTCP:LISTEN` + `TcpListener::bind` | native (OpenGL/WGPU) |

## Intégration Switch

MasterEnv est lancé en phase bloquante par le script `startup.sh` de Switch (`cargo run --release -- --path <racine>`) pour synchroniser les variables avant tout démarrage de service.
