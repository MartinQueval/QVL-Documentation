# ProjectCenter

> Portail d'entrée de l'écosystème QVL — SPA React autonome, **sans backend**, qui présente les projets et affiche leurs documentations.

![stack](https://img.shields.io/badge/React%2019%20·%20Vite%207%20·%20TypeScript%205.9-1f6feb)
![router](https://img.shields.io/badge/react--router-v7-ca4245)
![design%20system](https://img.shields.io/badge/canopui-baseline%20%5E2.2.0%20%C2%B7%20CI%20latest-4b5e40)
![deploy](https://img.shields.io/badge/qvl--project.com%20(apex)-serve--vitrine%20%C2%B7%20cloudflared-6f42c1)

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

Les pages de documentation ne sont pas embarquées dans le build : elles sont **récupérées au runtime, côté navigateur**, depuis le dépôt public `QVL-Documentation` via l'**API GitLab v4** (projet `84403403`, passé public le 2026-07-16).

- L'URL raw est construite par `buildGitlabRawUrl` (`src/lib/gitlabDocs.ts`), qui `encodeURIComponent` le chemin complet (les `/` deviennent `%2F`) et compose `{VITE_DOCS_API_PROJECT_URL}/repository/files/{chemin encodé}/raw?ref={VITE_DOCS_REF}`.
- Le hook `useProjectDoc(docPath)` fait le `fetch` sur cette URL, gère les états `loading` / `error` (`notFound` sur 404, `network` sinon, avec `retry`) / `success`, et annule proprement via `AbortController`. Le même builder sert `ProjectLogo` (logos servis depuis le repo doc).
- Le Markdown récupéré est ensuite rendu (sanitisé) par les composants `ProjectDocView` / `ProjectDocBody`. Les liens/images relatifs sont résolus vers des URL API v4 valides (`resolveRelativeDocPath` + `createDocUrlTransform`).
- La source est portée par deux variables d'environnement :

```
VITE_DOCS_API_PROJECT_URL=https://gitlab.com/api/v4/projects/84403403
VITE_DOCS_REF=main
```

### Décision — source des docs distantes (ADR `docs-source-gitlab-api`)

L'approche initiale (miroir GitHub raw via `VITE_DOCS_BASE_URL`, ADR `cors-docs-source`) a été **abandonnée** : depuis que le repo `QVL-Documentation` est **public** (2026-07-16), la doc est servie directement par l'**API GitLab v4**, ce qui supprime la dépendance au miroir GitHub. Le spike a validé le **CORS réel** (curl, en-têtes bruts, origine cross-origin) :

| Source | CORS | Contenu servi | Verdict |
|---|---|---|---|
| **API GitLab v4** (projet public) | **OK (`*`)** | **`200` ; forme `.../files/{encodé}/raw?ref=main`** | **Retenue** |
| Route web GitLab `/-/raw/` | KO (302 sans en-tête) | — | Écartée |
| Miroir GitHub raw | OK (`*`) | `200` par concaténation `base + docPath` | Ancienne source, remplacée |

L'API v4 impose une forme d'URL avec chemin encodé et suffixe `?ref=` (pas une simple concaténation) ; la construction est donc centralisée dans `src/lib/gitlabDocs.ts`. Voir l'ADR `docs/decisions/docs-source-gitlab-api.md` (qui remplace `cors-docs-source.md`, conservé pour historique).

## Design system : canopui

ProjectCenter consomme [**canopui**](../QVL-CanopUI/README.md) depuis le [registre Verdaccio privé](../hebergement/registre-npm.md) suivant un modèle **« auto-pull + build sur `canopui@latest` »** (et non une version épinglée exacte). La baseline committée dans `package.json` est `canopui@^2.2.0`, mais la CI exécute `npm install canopui@latest` avant chaque `npm run build` : le build déployé embarque donc toujours la **dernière** CanopUI publiée (aujourd'hui `canopui@3.0.1`). En développement, la lib peut aussi être consommée via un **tarball local** (`npm run canopui:local`, `canopui.local.tgz` gitignoré) : chaque pack injecte une version prerelease unique `X.Y.Z-local.<timestamp>` pour contourner le cache npm. Un garde-fou (`tools/check-lock-no-local.sh`, appelé par un hook `pre-commit` versionné dans `.githooks` **et** par le job CI `lock-guard`) **rejette** tout `package-lock.json` épinglé en `-local`.

## Déploiement

- **SPA React servie à l'APEX `qvl-project.com`** — l'app est servie par `serve-vitrine.mjs` (**tâche Windows** `QVL-ProjectCenter`, port **8300**) derrière **cloudflared**. Il n'y a **pas** de vhost `projectcenter.qvl-project.com` ni de serveur nginx : ProjectCenter occupe l'apex du domaine.
- **Déploiement par CI/CD GitLab** (`.gitlab-ci.yml`, runner shell local WSL Ubuntu-24.04). Sur `main`, le job `build` valide l'artefact, puis `deploy` **rebuild** (avec `npm install canopui@latest`) et copie atomiquement (`.new` puis swap) `dist/` vers `/mnt/c/QVL/deploy/projectcenter` (= `C:\QVL\deploy\projectcenter`) ; **pas d'artefact GitLab porté** (le deploy tourne sur la même machine). Un job `update-checkout` fait ensuite un `git pull --ff-only` du clone local persistant. L'ancien `npm run deploy:local` n'est plus le chemin de déploiement.
- **Deep-links** — l'app utilise `createBrowserRouter` ; le service d'apex renvoie sur `index.html`, donc pas de repli HashRouter nécessaire.
- **CSP en défense en profondeur** — même politique portée à deux niveaux : meta tag injecté au **build** (plugin Vite `projectcenter-csp-meta`, `apply: "build"`) et **header HTTP** côté hébergement. Directives clés (Gate sécu 2) : `default-src 'self'` ; `connect-src 'self' https://gitlab.com` (fetch docs via l'API GitLab v4) ; `img-src 'self' data: https://gitlab.com https://img.shields.io` (images des README + badges shields.io) ; `style-src 'self' 'unsafe-inline'` (MUI/emotion injectent leurs styles inline) ; `font-src 'self' data:` ; `script-src 'self'` ; `base-uri 'self'` ; `form-action 'self'` ; `object-src 'none'`. Le header HTTP ajoute `frame-ancestors 'none'` (ignoré en meta par le navigateur), `X-Content-Type-Options: nosniff`, `Referrer-Policy` et `server_tokens off`.
- **Fichiers `deploy/nginx/*.conf` conservés mais superseded** — le repo garde encore `deploy/nginx/projectcenter.qvl-project.com.conf` de l'itération nginx d'origine ; il est **remplacé** par le service d'apex `serve-vitrine` + cloudflared et n'est plus utilisé.
- **Police Chivo (Google Fonts) volontairement bloquée** — les hôtes Google ne sont pas whitelistés ; Chivo retombe sur la stack système sans casse fonctionnelle. Cible visée : self-host de Chivo dans canopui.

## Environnement & démarrage

- Registre privé configuré via `.npmrc`.
- Variables clés : `VITE_DOCS_API_PROJECT_URL` (projet GitLab de la doc) et `VITE_DOCS_REF` (branche). Voir `docs/decisions/docs-source-gitlab-api.md`.
- Démarrage local orchestré via l'outil **Switch** du méta-workspace [QVL-Studio](../QVL-Studio/README.md) (dossier `Tools/`).

## Liens utiles

- Design system : [CanopUI](../QVL-CanopUI/README.md)
- Registre npm : [guide du registre](../hebergement/registre-npm.md)
- Pipeline CI/CD : [documentation pipeline](../pipeline/README.md)
- Méta-workspace : [QVL-Studio](../QVL-Studio/README.md)

---

_Documentation maintenue par la flotte QVL-Studio · [Hub de documentation](../README.md)_
