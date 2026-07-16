# QVL-Studio

> **Méta-workspace** de développement de l'écosystème QVL-Studio — pas un projet applicatif.

QVL-Studio n'est pas une application : c'est un **dossier de travail** qui regroupe, côte à côte, les **checkouts** des dépôts QVL-Studio et l'outillage transverse nécessaire pour les faire tourner ensemble en local.

## Contenu

| Élément | Rôle |
|---|---|
| `CanopUI/` | Checkout du design system — voir [CanopUI](../QVL-CanopUI/README.md) |
| `ProjectCenter/` | Checkout du portail d'entrée — voir [ProjectCenter](../QVL-ProjectCenter/README.md) |
| `Documentation/` | Checkout du hub de documentation (ce dépôt) |
| `Tools/` | Outillage transverse : **MasterEnv** et **Switch** |
| `.masterenv` | Fichier central des variables d'environnement, généré et synchronisé par MasterEnv |

## Outillage (`Tools/`)

- **MasterEnv** — gestionnaire de variables d'environnement multi-repo (Rust). Il scanne les dépôts voisins, centralise leurs variables dans `.masterenv`, détecte les conflits et les collisions de ports, et propose une GUI de résolution. La synchronisation propage les valeurs vers chaque fichier de config des dépôts (`.env`, `.toml`…) sans jamais créer de clé absente.
- **Switch** — orchestrateur de démarrage local des applications du workspace (ex. ProjectCenter).

## À noter

Toute l'activité de développement, de versioning et de CI/CD se fait **dans les dépôts individuels**, chacun miroité et outillé indépendamment. Le méta-workspace ne fait que les rassembler pour le confort du poste de dev.

---

_Documentation maintenue par la flotte QVL-Studio · [Hub de documentation](../README.md)_
