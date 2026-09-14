# CH-Portail-Admin

Portail d'administration de l'écosystème **CustHome** : tableau de bord de supervision, gestion des comptes utilisateurs (validation, statut, rôles, mot de passe, liste des appareils reconnus) et gestion du catalogue de rôles par portail. Réservé aux administrateurs.

![React](https://img.shields.io/badge/React-19-61dafb) ![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6) ![Vite](https://img.shields.io/badge/Vite-7-646cff) ![canopui](https://img.shields.io/badge/canopui-latest-lightgrey) ![Express](https://img.shields.io/badge/Express-5-000000)

## Stack

- **React 19** + **TypeScript** (strict), routage via **react-router-dom 7**
- **canopui** (UI library QVL) — `PageScaffold`, `DataTable`, `SidePanel`, `Toggle`, `CardGrid`…
- **MUI 7** + Emotion (socle de canopui)
- Build **Vite 7**, tests **Vitest**
- Serveur de production **Express** (`server.js` + `app.js`) qui sert le build et proxifie `/api`

## Port par défaut

| Contexte | Port |
| --- | --- |
| Dev (Vite, `.env` `PORT`) | **3201** |
| Prod (`server.js`, fallback `PORT`) | 3001 |

> Le port documenté de référence est **3201** (`.env.example`, `vite.config.ts`).

## Routes / pages

Toutes les routes (hors `/forbidden`) sont protégées par la garde **`RequireAdmin`** puis rendues dans `AdminLayout` (`PageScaffold` avec navigation Dashboard / Utilisateurs / Rôles). `/` et les routes inconnues redirigent vers `/dashboard`.

| Chemin | Page | Garde (rôle requis) | Description |
| --- | --- | --- | --- |
| `/forbidden` | `Forbidden` | Aucune | Accès refusé + bouton pour changer de compte (retour login) |
| `/` | — | `RequireAdmin` (`admin`) | Redirige vers `/dashboard` |
| `/dashboard` | `Dashboard` | `RequireAdmin` (`admin`) | Cartes de statistiques par statut (`DashboardStats`), interrupteur d'activation des inscriptions (`RegistrationToggleCard`), carte de trafic (`TrafficCard`) et carte des comptes en attente de validation (`PendingUsersCard`, affichée uniquement s'il en existe : approuver / supprimer) |
| `/users` | `Users` | `RequireAdmin` (`admin`) | Table de tous les utilisateurs (recherche, statut, rôles portail) ; panneau latéral d'édition : profil, mot de passe, rôles (`UserRolesEditor`), liste des appareils reconnus (`UserDevicesList`) avec révocation d'un appareil et restriction aux appareils reconnus (`whitelist_only`), activation/désactivation, suppression |
| `/roles` | `Roles` | `RequireAdmin` (`admin`) | Grille de cartes par portail (`admin`, `drive`, `budgy`, `home`) permettant d'ajouter/supprimer des sous-rôles |
| `*` | — | `RequireAdmin` (`admin`) | Redirige vers `/dashboard` |

## Flux d'authentification

- La garde **`RequireAdmin`** s'appuie sur le composant `RouteGuard` de canopui : elle appelle `getMe` (`/auth/me`) et vérifie la présence du rôle **`admin`** (`isPortalAdmin`).
- En cas de **401** (session absente/expirée), redirection vers le portail Authenticator via `loginUrl()` (dépôt du cookie `ch_redirect` + intention de redirection).
- Si l'utilisateur est authentifié **mais sans le rôle** `admin`, affichage de `/forbidden`.
- Les cookies de session (`ch_token` / `ch_refresh`, HttpOnly) restent same-origin grâce au proxy `/api` du serveur Express.
- La déconnexion (`logout` → `/auth/logout`) renvoie ensuite vers le login.

## API consommée

Client canopui `createApiClient({ basePath: "/api", withRefresh: true })` ; deux familles de préfixes via la gateway : **`/api/auth`** (session) et **`/api/admin`** (administration).

| Domaine | Endpoints (extraits) |
| --- | --- |
| Session | `/auth/me`, `/auth/logout` |
| Utilisateurs | `/admin/users` (GET, liste paginée + filtres), `/admin/users/:id` (PUT/DELETE), `/admin/users/:id/status`, `/admin/users/:id/roles`, `/admin/users/:id/password`, `/admin/users/:id/whitelist` (PUT, bascule le flag `whitelist_only`), `/admin/users/:id/devices/:deviceId` (DELETE, révoque un appareil reconnu) |
| Paramètres | `/admin/settings/registration` (GET/PUT) |
| Analytics | `/admin/analytics/traffic?period=` (day/week/month/year) |
| Rôles | `/admin/roles` (GET/POST), `/admin/roles/:id` (DELETE) |

Voir la documentation de l'API : [../CH-Api-Authenticator/README.md](../CH-Api-Authenticator/README.md) (l'administration est portée par l'API Authenticator) — routée par la [gateway](../CH-Api-GateWay/README.md).

## Configuration (`.env`)

| Variable | Rôle | Exemple |
| --- | --- | --- |
| `PORT` | Port d'écoute du serveur (dev Vite / prod Express) | `3201` |
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

- **Garde à deux niveaux implicite** : `RequireAdmin` (via `RouteGuard`) distingue le cas non authentifié (401 → login) du cas authentifié sans rôle (`/forbidden`).
- **Catalogue de rôles par portail** : les rôles sont organisés par portail (`admin`, `drive`, `budgy`, `home`) et par nature (`portal` / `sub`) ; la page Rôles ne gère que les sous-rôles.
- **Gestion fine des comptes** : validation des comptes en attente, activation/désactivation, réinitialisation de mot de passe, gestion des appareils reconnus (révocation, restriction via le flag `whitelist_only`), attribution de rôles depuis un panneau latéral.
- **Dépendance canopui non épinglée en pratique** : `package.json` déclare une borne (`^2.2.0`), mais le job `build` de la CI exécute `npm install canopui@latest` avant de builder — l'artefact déployé embarque toujours la dernière version publiée (d'où le badge `canopui-latest`).

## Incohérences relevées

- Le fallback de port dans `server.js` est **3001**, alors que `.env.example` et `vite.config.ts` utilisent **3201**. En pratique `PORT` est fourni via l'environnement.
- Le fallback `GATEWAY_URL` dans `server.js` est `http://localhost:8080`, alors que `.env.example` et `vite.config.ts` utilisent `http://localhost:8180`.
