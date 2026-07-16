# CH-Portal-Drive

Portail de stockage de fichiers de l'écosystème **CustHome** : explorateur de fichiers et dossiers (upload, import de dossier, renommage, déplacement, corbeille), galerie média groupée par mois avec visionneuse, et console d'administration des quotas. Réservé aux utilisateurs disposant du rôle Drive.

![React](https://img.shields.io/badge/React-19-61dafb) ![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6) ![Vite](https://img.shields.io/badge/Vite-7-646cff) ![canopui](https://img.shields.io/badge/canopui-1.0.1-lightgrey) ![Express](https://img.shields.io/badge/Express-5-000000)

## Stack

- **React 19** + **TypeScript** (strict), routage via **react-router-dom 7**
- **canopui** (UI library QVL) — `PageScaffold`, `DataTable`, `SidePanel`, `ProgressBar`, `ConfirmDialog`…
- **MUI 7** + Emotion (socle de canopui)
- Build **Vite 7**, tests **Vitest**
- Serveur de production **Express** (`server.js` + `app.js`) qui sert le build et proxifie `/api`

## Port par défaut

| Contexte | Port |
| --- | --- |
| Dev (Vite, `.env` `PORT`) | **3202** |
| Prod (`server.js`, fallback `PORT`) | 3002 |

> Le port documenté de référence est **3202** (`.env.example`, `vite.config.ts`).

## Routes / pages

Toutes les routes (hors `/forbidden`) sont protégées par la garde **`RequireDrive`** puis rendues dans `DriveLayout` (`PageScaffold` avec navigation Fichiers / Galerie / Corbeille — et Admin si rôle) et un `StorageBar` en widget de barre latérale. `/` et les routes inconnues redirigent vers `/files`.

| Chemin | Page | Garde (rôle requis) | Description |
| --- | --- | --- | --- |
| `/forbidden` | `Forbidden` | Aucune | Accès refusé |
| `/` | — | `RequireDrive` (`drive`) | Redirige vers `/files` |
| `/files` | `Files` | `RequireDrive` (`drive`) | Explorateur : fil d'Ariane, recherche, vues liste/grille, upload de fichiers, import de dossier, création de dossier, renommage, déplacement, propriétés, mise à la corbeille, sélection multiple, glisser-déposer |
| `/trash` | `Files` (`trash`) | `RequireDrive` (`drive`) | Corbeille : restauration, purge définitive (unitaire ou vidage complet), propriétés |
| `/gallery` | `Gallery` | `RequireDrive` (`drive`) | Galerie média groupée par mois, vignettes, badge vidéo, ouverture en visionneuse (`Lightbox`) |
| `/admin` | `Admin` | `RequireDrive` + `RequireDriveAdmin` (`drive_admin`) | Table des utilisateurs Drive : consommation vs quota (`ProgressBar`), édition du quota (Gio), recalcul de l'usage |
| `*` | — | `RequireDrive` (`drive`) | Redirige vers `/files` |

## Flux d'authentification

- La garde **`RequireDrive`** (via `RouteGuard` de canopui) appelle `getMe` (`/auth/me`) et vérifie le rôle **`drive`** (`isPortalDrive`).
- **401** → redirection vers le portail Authenticator via `loginUrl()` (cookie `ch_redirect` + intention de redirection). Authentifié sans rôle `drive` → `/forbidden`.
- La garde imbriquée **`RequireDriveAdmin`** protège `/admin` : elle lit l'utilisateur courant du contexte (`useCurrentUser`) et vérifie le rôle **`drive_admin`** (`isDriveAdmin`) ; sinon redirection vers `/files`. L'entrée « Admin » de la navigation n'apparaît que pour ce rôle.
- Cookies de session (`ch_token` / `ch_refresh`, HttpOnly) same-origin via le proxy `/api`. La déconnexion (`/auth/logout`) renvoie au login.

## API consommée

Client canopui `createApiClient({ basePath: "/api", withRefresh: true })` ; préfixes via la gateway : **`/api/auth`** (session) et **`/api/drive`** (fichiers, stockage, admin).

| Domaine | Endpoints (extraits) |
| --- | --- |
| Session | `/auth/me`, `/auth/logout` |
| Stockage | `/drive/me/storage` |
| Navigation | `/drive/files` (GET liste + `?parent=`, POST upload), `/drive/folders` (POST), `/drive/search?q=`, `/drive/duplicates` |
| Nœuds | `/drive/nodes/:id` (PATCH renommage/déplacement, DELETE purge), `/drive/nodes/:id/trash`, `/drive/nodes/:id/restore` |
| Corbeille | `/drive/trash`, `/drive/trash/purge` |
| Galerie | `/drive/gallery`, `/drive/files/:id/thumbnail`, `/drive/files/:id/content` |
| Admin | `/drive/admin/users` (GET), `/drive/admin/users/:id` (PATCH quota), `/drive/admin/users/:id/recompute` |

Voir la documentation de l'API : [../CH-Api-Drive/README.md](../CH-Api-Drive/README.md) et [../CH-Api-Authenticator/README.md](../CH-Api-Authenticator/README.md) — routées par la [gateway](../CH-Api-GateWay/README.md).

## Configuration (`.env`)

| Variable | Rôle | Exemple |
| --- | --- | --- |
| `PORT` | Port d'écoute du serveur (dev Vite / prod Express) | `3202` |
| `GATEWAY_URL` | URL de la gateway vers laquelle `/api` est proxifié | `http://localhost:8180` |
| `VITE_AUTH_PORTAL_URL` | URL du portail Authenticator (redirection si session absente/expirée) | `http://localhost:3200` |

## Scripts npm

| Script | Action |
| --- | --- |
| `npm run dev` | Serveur de dev Vite (`--force`) |
| `npm run build` | `tsc -b` puis build Vite |
| `npm run preview` | Prévisualisation du build |
| `npm run lint` | ESLint |
| `npm test` | Vitest (run unique) |
| `npm run test:watch` | Vitest en watch |
| `npm run coverage` | Couverture (seuils 80 %) |
| `npm start` | Serveur Express de production (`node server.js`) |

## Particularités

- **Deux rôles** : `drive` (accès au portail) et `drive_admin` (console d'administration des quotas via `RequireDriveAdmin`).
- **Corbeille** : la même page `Files` sert `/files` et `/trash` (prop `trash`), avec actions différenciées (restauration / purge / vidage complet).
- **Galerie** : regroupement des médias par mois (à partir de `taken_at` ou `created_at`), vignettes servies par l'API, visionneuse `Lightbox`, badge pour les vidéos.
- **UX riche** : glisser-déposer de fichiers, import de dossiers (`webkitdirectory`), sélection multiple avec actions groupées, vues liste/grille persistées, `StorageBar` de suivi du quota dans la barre latérale.

## Incohérences relevées

- Le fallback de port dans `server.js` est **3002**, alors que `.env.example` et `vite.config.ts` utilisent **3202**. En pratique `PORT` est fourni via l'environnement.
- Le fallback `GATEWAY_URL` dans `server.js` est `http://localhost:8080`, alors que `.env.example` et `vite.config.ts` utilisent `http://localhost:8180`.
