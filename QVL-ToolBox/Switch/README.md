# Switch

## Rôle

Switch est un orchestrateur de démarrage et d'arrêt de la stack de services d'un projet. Il repose sur deux scripts Bash : `startup.sh` lance tous les services dans le bon ordre (avec vérification de santé), `shutdown.sh` les coupe proprement dans l'ordre inverse. Le tout est multi-plateforme (détection MinGW/MSYS/Cygwin pour Windows, sinon comportement POSIX).

## Stack

- Bash (compatible Git Bash / MSYS sous Windows).
- Dépendances externes selon les vérifications activées : `curl` (health-check HTTP), et les clients de base de données optionnels (`mongosh`, `pg_isready`, `mysqladmin`, `redis-cli`), avec repli sur un test TCP brut.

## startup.sh

```bash
./startup.sh          # mode production (défaut)
./startup.sh --dev    # mode développement
```

Le mode (`prod` ou `dev`) sélectionne, pour chaque service, la commande de lancement correspondante. La racine du projet est le parent du dossier Switch ; les logs sont écrits dans `.startup-logs/`.

Déroulé en quatre phases :

1. **MasterEnv (bloquant)** — synchronisation des variables d'environnement via `cargo run --release -- --path <racine>` avant tout démarrage.
2. **Vérification des bases de données** — chaque base critique est testée (client natif selon le type `mongo`/`postgres`/`mysql`/`redis`, ou repli TCP) ; un échec critique arrête tout. Les bases non-critiques indisponibles émettent un avertissement sans bloquer.
3. **Services critiques** — lancés puis attendus jusqu'à réponse de leur URL de santé (dans la limite du timeout par service). En production, la librairie UI est buildée (pas de process long-running). Si un service critique ne démarre pas, tous les services sont arrêtés.
4. **Services non-critiques** — lancés ; un échec émet un avertissement sans bloquer.

Le script reste ensuite en attente ; `Ctrl+C` déclenche l'arrêt de tous les services démarrés.

### Définition d'un service

Les services sont déclarés dans deux tableaux (`CRITICAL_SERVICES`, `NON_CRITICAL_SERVICES`) au format `nom|répertoire|commande_dev|commande_prod|url_health|timeout_sec`. Les bases sont déclarées au format `type|label|host|port|uri`.

## shutdown.sh

```bash
./shutdown.sh          # arrêt gracieux (10 s max par service)
./shutdown.sh --force  # kill immédiat
```

Arrête les services dans l'ordre inverse du démarrage, à partir du tableau `SERVICES` au format `nom|process_pattern|port`. Le PID est retrouvé par le port en écoute (`netstat` sous Windows, `lsof` sinon). En mode gracieux, un `SIGTERM` (ou `taskkill`) est envoyé puis, passé le délai `GRACE_TIMEOUT` (10 s), un kill forcé est appliqué. Le script termine sur un résumé (`arrêtés` / `déjà off` / `en échec`) et un code de sortie non nul si au moins un arrêt a échoué.

## Configuration

Il n'y a pas de fichier de configuration séparé : les services, bases de données et ports sont édités directement dans les tableaux en tête des deux scripts. Le dossier Switch est attendu à la racine du projet, à côté du dossier MasterEnv qu'il invoque en phase 1.
