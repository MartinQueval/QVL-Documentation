# ProjectCenter

> Portail d'entrée de l'écosystème QVL — SPA React autonome, **sans backend**, qui présente les projets et affiche leurs documentations.

![stack](https://img.shields.io/badge/React%2019%20·%20Vite%207%20·%20TypeScript%205.9-1f6feb)
![router](https://img.shields.io/badge/react--router-v7-ca4245)
![design%20system](https://img.shields.io/badge/canopui-1.1.0-4b5e40)
![deploy](https://img.shields.io/badge/projectcenter.qvl--project.com-nginx-6f42c1)

## Rôle

ProjectCenter est le **point d'entrée unique** vers l'ensemble des projets QVL. C'est une application monopage (SPA) construite avec **Vite 7**, **React 19** et **TypeScript 5.9** en mode strict, consommant le design system [**canopui**](../QVL-CanopUI/README.md). Elle n'a **pas de backend** : le catalogue de projets est statique et les documentations sont chargées au runtime depuis un dépôt public.

## Pages & routes

Le routage utilise `createBrowserRouter` (react-router v7). Toutes les routes sont imbriquées sous le layout `App` :

| Route | Page | Rôle |
|---|---|---|
| `/` | `HomePage` | Accueil, une section par organisation QVL, cards projet en carrousel |
| `/:section` | `SectionPage` | Détail d'une section et de ses projets |
| `/:section/:project` | `ProjectDocPage` | Documentation d'un projet, rendue en Markdown |
| `*` | — | Redirection vers `/` |

Les **sections** (QVL-Studio, QVL-Hobbies, QVL-ToolBox, QVL-CustHome) et les **projets** (nom, description, logo, URL, chemin de doc) sont décrits statiquement dans `src/data/projects.ts`. Une **card projet** présente titre / description / logo ; un clic ouvre l'URL du projet.

## Chargement des docs au runtime

Les pages de documentation ne sont pas embarquées dans le build : elles sont **récupérées au runtime, côté navigateur**, depuis le **miroir GitHub public raw** du dépôt `Documentation`.

- Le hook `useProjectDoc(docPath)` fait un `fetch` sur `new URL(docPath, VITE_DOCS_BASE_URL)`, gère les états `loading` / `error` (`notFound` sur 404, `network` sinon, avec `retry`) / `success`, et annule proprement via `AbortController`.
- Le Markdown récupéré est ensuite rendu (sanitisé) par les composants `ProjectDocView` / `ProjectDocBody`.
- La base d'URL est portée par la variable d'environnement **`VITE_DOCS_BASE_URL`** :

```
VITE_DOCS_BASE_URL=https://raw.githubusercontent.com/QVL-Studio/Documentation/main/
```

### Décision — source des docs distantes (ADR `cors-docs-source`)

Le choix de la source résulte d'un spike (SCRUM-312) qui a testé le **CORS réel** (curl, en-têtes bruts, `Origin` de dev) de trois candidates :

| Source | CORS | Contenu servi | Verdict |
|---|---|---|---|
| API GitLab raw v4 | OK (`*`) | `404` + forme d'URL incompatible (`?ref=`, chemin encodé) | Écartée |
| Route web GitLab `/-/raw/` | KO (302 sans en-tête) | — | Écartée |
| **Miroir GitHub raw** | **OK (`*`)** | **`200`, concaténation `base + docPath`** | **Retenue** |

Le miroir GitHub raw est la seule source simultanément fetchable en navigateur, servant réellement le contenu, et utilisable par simple concaténation `VITE_DOCS_BASE_URL + docPath`.

## Design system : canopui

ProjectCenter épingle la version **hébergée exacte** `canopui@1.1.0` depuis le [registre Verdaccio privé](../hebergement/registre-npm.md) (US6 / SCRUM-318). En développement, la lib peut aussi être consommée via un **tarball local** (`npm run canopui:local`, `canopui.local.tgz` gitignoré) : chaque pack injecte une version prerelease unique `X.Y.Z-local.<timestamp>` pour contourner le cache npm. Un hook `pre-commit` versionné (`.githooks`) **rejette** tout commit d'un `package-lock.json` épinglé en `-local` — la baseline committée reste `canopui@X.Y.Z`.

## Déploiement

- **Hôte : nginx local QVL**, le même serveur que `npm.qvl-project.com` (Verdaccio) et `canopui.qvl-project.com` (vitrine). Vhost `projectcenter.qvl-project.com`, artefacts servis depuis `C:\QVL\deploy\projectcenter`, déploiement via `npm run deploy:local` (copie atomique).
- **Deep-links** — l'app utilise `createBrowserRouter` ; nginx renvoie sur `index.html` (`try_files $uri $uri/ /index.html;`), donc pas de repli HashRouter nécessaire.
- **CSP en défense en profondeur** — même politique portée à deux niveaux : meta tag injecté au **build** (plugin Vite, `apply: "build"`) et **header HTTP** délivré par nginx. Directives clés (Gate sécu 2) : `default-src 'self'` ; `connect-src 'self' https://raw.githubusercontent.com` (fetch docs) ; `img-src` autorisant `raw.githubusercontent.com` et `img.shields.io` (badges) ; `style-src 'self' 'unsafe-inline'` (MUI/emotion injectent leurs styles inline) ; `script-src 'self'`. Le header nginx ajoute `frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy` et `server_tokens off`.
- **Police Chivo (Google Fonts) volontairement bloquée** — les hôtes Google ne sont pas whitelistés ; Chivo retombe sur la stack système sans casse fonctionnelle. Cible visée : self-host de Chivo dans canopui.

## Environnement & démarrage

- Registre privé configuré via `.npmrc`.
- Variable clé : `VITE_DOCS_BASE_URL`.
- Démarrage local orchestré via l'outil **Switch** du méta-workspace [QVL-Studio](../QVL-Studio/README.md) (dossier `Tools/`).

## Liens utiles

- Design system : [CanopUI](../QVL-CanopUI/README.md)
- Registre npm : [guide du registre](../hebergement/registre-npm.md)
- Pipeline CI/CD : [documentation pipeline](../pipeline/README.md)
- Méta-workspace : [QVL-Studio](../QVL-Studio/README.md)

---

_Documentation maintenue par la flotte QVL-Studio · [Hub de documentation](../README.md)_
