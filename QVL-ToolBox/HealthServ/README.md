# HealthServ

## Rôle

HealthServ est un service de maintenance système écrit en Rust, destiné aux machines laissées allumées en continu. Il automatise trois tâches d'entretien : lancement automatique de services après un redémarrage, redémarrage planifié quotidien de la machine, et purge périodique du cache RAM. Il n'expose **aucune API HTTP** : c'est un processus local piloté par un fichier de configuration.

## Stack

- Rust.
- Support multi-plateforme : Windows, Linux, macOS. La cible OS est déclarée dans la configuration (`os`).

## Fonctionnement

Au démarrage, HealthServ charge `config.toml` (chemin relatif `config.toml`), puis :

1. lance les services listés (`launch_services`) ;
2. arme le redémarrage quotidien à l'heure configurée (`restart`) ;
3. entre dans une boucle qui purge le cache RAM à l'intervalle configuré (`clear_ram`).

Si le fichier de configuration est absent ou invalide, le démarrage échoue immédiatement.

## Configuration

Copier `config.toml.example` en `config.toml` (jamais committé) et l'adapter. Le fichier n'est jamais publié.

| Clé | Type | Rôle |
|---|---|---|
| `os` | chaîne | Système cible : `"windows"`, `"linux"` ou `"mac"`. |
| `restart_hour` | chaîne | Heure du redémarrage quotidien, format `"HH:MM"` sur 24 h (ex. `"04:00"`). |
| `cache_clear_interval` | entier | Intervalle de purge RAM, en heures (entier, ex. `6`). |
| `services` | liste de chaînes | Chemins complets vers les exécutables à relancer après un redémarrage. |

Exemple :

```toml
os = "windows"
restart_hour = "04:00"
cache_clear_interval = 6
services = [
    "C:\\path\\to\\service1.exe",
    "C:\\path\\to\\service2.exe",
]
```

Toute modification prend effet au prochain redémarrage automatique.

## Démarrage automatique selon l'OS

- **Windows** : ajouter un raccourci vers l'exécutable dans le dossier de démarrage (`shell:startup`).
- **Linux** : ajouter `@reboot /chemin/vers/HealthServ` à la crontab (`crontab -e`). La purge RAM nécessite les droits sudo sans mot de passe sur `/bin/sh` (via `visudo`, ligne `NOPASSWD: /bin/sh`).
- **macOS** : support prévu (procédure non détaillée dans la source).

## Notes

Les chemins de services sous Windows doivent doubler les antislash (`\\`). Le format de `restart_hour` doit rester une chaîne entre guillemets ; `cache_clear_interval` doit rester un entier nu (sans guillemets).
