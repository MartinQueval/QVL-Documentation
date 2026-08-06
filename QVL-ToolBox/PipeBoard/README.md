# PipeBoard

## Rôle

PipeBoard est un dashboard centralisé, en lecture seule, des pipelines de gitlab.com, organisés par group et par repo. Le front React interroge un backend Express qui interroge lui-même l'API GitLab avec un Personal Access Token (scope `read_api` uniquement), met les résultats en cache et les rafraîchit par polling. Cette page documente le **backend Express** ; l'interface est bâtie sur React/Vite/canopui.

## Stack

- Front : React + TypeScript + Vite, design system canopui.
- Back : Express (TypeScript strict) exécuté via `tsx`. Les imports relatifs entre modules serveur portent l'extension `.ts` explicite.
- Node.js >= 20.

## Ports

- Front (Vite) : `5190`.
- Back (Express) : `5191`.

L'hôte du backend est forcé sur `127.0.0.1` (jamais `0.0.0.0`). Le front proxifie `/api` vers `http://127.0.0.1:5191`. En production, le backend sert aussi le build statique du front depuis `dist/`.

## Hébergement et déploiement

Production : **https://tb-pipeboard.qvl-project.com** — service systemd `tb-pipeboard` (WSL, utilisateur `qvlsvc`, `WorkingDirectory=/opt/qvl-toolbox/pipeboard`), exposé via le Cloudflare Tunnel `qvl-infra`.

**Seul projet de la flotte hébergé sur GitHub** (`github.com/QVL-ToolBox/PipeBoard`) et **sans CI** : le runner GitLab ne peut pas l'atteindre. Le déploiement est manuel, par script, sur le serveur QVL :

```bash
sudo tb-deploy-pipeboard              # source par défaut : /mnt/c/QVL/QVL-ToolBox/PipeBoard
```

Le script copie les sources, installe, construit sur la **dernière CanopUI publiée** (`npm install canopui@latest`), bascule atomiquement dans `/opt/qvl-toolbox/pipeboard` et redémarre le service — avec restauration automatique de la version précédente si le service ne repart pas.

Deux contraintes qu'il porte :

- **Le build ne peut pas se faire sur `/mnt/c`** : npm y échoue sur `chmod` (`EPERM`, limite du FS Windows vu depuis WSL). Le script construit dans un répertoire natif WSL.
- **Les `devDependencies` sont nécessaires au runtime** (`npm start` lance le serveur via `tsx`) : ne jamais élaguer avec `npm prune --omit=dev`.

> Contrepartie de l'absence de CI : la production dérive silencieusement. Elle est restée sur `canopui@1.0.1` alors que la migration 2.x était committée depuis longtemps — écart détecté et corrigé le 2026-08-06 (passage direct en `2.3.0`). Relancer le script après toute évolution du design system.

## Configuration

Aucun fichier de configuration n'est requis. Les groups surveillés sont découverts automatiquement après enregistrement du token, via les groups dont le porteur du token est membre (`GET /groups?membership=true` côté GitLab). Les sous-groups descendants sont écartés car le listing des projets est déjà récursif. Le registre npm QVL est configuré dans `.npmrc` (lecture anonyme) ; installation reproductible via `npm ci`.

Le token est conservé côté serveur en mémoire et dupliqué dans un cookie de session `pipeboard_token` (`httpOnly`, `sameSite=strict`, `path=/api`, `maxAge` 30 jours). Après un redémarrage du client, si la mémoire serveur ne contient plus de token mais que le cookie est présent, le token est restauré et le polling réarmé. Un token rejeté par GitLab (401/403) est mémorisé en mémoire seule : le cookie correspondant est alors effacé sans nouvelle tentative de restauration.

## Endpoints

Toutes les routes sont servies sous le préfixe `/api` (même origine que le front). Aucun header d'authentification n'est exigé : l'identité GitLab est portée par le PAT enregistré côté serveur (mémoire + cookie).

| Méthode | Chemin | Auth | Description | Codes |
|---|---|---|---|---|
| GET | `/api/status` | Non | État courant : `tokenSet`, `lastListingRefresh`, `lastStatusRefresh`, `rateLimited`. | `200` |
| GET | `/api/pipelines` | Non | Dernier état connu du cache (arbre groups → repos → pipeline). | `200` |
| POST | `/api/token` | Non | Enregistre le PAT `{ "token": string }`, pose le cookie et arme le polling. | `200`, `400` |
| POST | `/api/refresh` | Non | Déclenche immédiatement un cycle listing + statuts (fire-and-forget). | `200`, `400` |
| DELETE | `/api/token` | Non | Purge le token, arrête le polling, vide le cache et efface le cookie. | `200` |
| ALL | `/api/*` (non résolu) | Non | Toute autre route sous `/api` non gérée. | `404` |

Précisions sur les codes :

- `POST /api/token` renvoie `400` si le payload ne contient pas de champ `token` chaîne non vide.
- `POST /api/refresh` renvoie `400` si aucun token n'est configuré.
- Un corps JSON syntaxiquement invalide sur une route `/api` est intercepté par le gestionnaire d'erreurs et renvoie `400`.
- Une exception non gérée renvoie `500`.

## Format d'erreur

Les réponses de succès sont du JSON (`{ "ok": true }` pour les mutations, un objet d'état pour les lectures). Les erreurs sont du JSON de forme :

```json
{ "error": "Invalid token payload" }
```

Messages observés : `Invalid token payload` (`400`), `No token configured` (`400`), `Invalid JSON body` (`400` sur JSON malformé), `Not Found` (`404` sur `/api/*` non résolu), `Internal Server Error` (`500`).
