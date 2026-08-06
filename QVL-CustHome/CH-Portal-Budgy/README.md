# CH-Portal-Budgy

Portail de gestion budgétaire personnel de l'écosystème **CustHome** : rattachement de comptes bancaires via le consentement PSD2/AIS, tableau de bord (soldes, reste à dépenser, prévisionnel), consultation et catégorisation des transactions, gestion et renouvellement des consentements. Réservé aux utilisateurs disposant du rôle Budgy.

![React](https://img.shields.io/badge/React-19-61dafb) ![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6) ![Vite](https://img.shields.io/badge/Vite-7-646cff) ![canopui](https://img.shields.io/badge/canopui-latest-lightgrey) ![Express](https://img.shields.io/badge/Express-5-000000)

## Stack

- **React 19** + **TypeScript** (strict), routage via **react-router-dom 7**
- **canopui** (UI library QVL) — `PageScaffold`, `CardGrid`, `Feedback`, `Spinner`…
- **MUI 7** + Emotion (socle de canopui)
- **mqtt** — relais temps réel optionnel (notifications / rechargement sur événement)
- Build **Vite 7**, tests **Vitest**
- Serveur de production **Express** (`server.js` + `app.js`) qui sert le build et proxifie `/api`

## Port par défaut

| Contexte | Port |
| --- | --- |
| Dev (Vite, `.env` `PORT`) | **3203** |
| Prod (`server.js`, fallback `PORT`) | 3203 |

## Routes / pages

Toutes les routes (hors `/forbidden`) sont protégées par la garde **`RequireBudgy`** puis rendues dans `BudgyLayout` (`PageScaffold` avec navigation Tableau de bord / Comptes / Catégories / Consentements, notifications et toast d'erreur de synchronisation). Les routes inconnues redirigent vers `/`.

| Chemin | Page | Garde (rôle requis) | Description |
| --- | --- | --- | --- |
| `/forbidden` | `Forbidden` | Aucune | Accès refusé |
| `/` | `Dashboard` | `RequireBudgy` (`budgy`) | Tableau de bord : soldes consolidés, reste à dépenser, budget prévisionnel |
| `/banque` | `RattacherBanque` | `RequireBudgy` (`budgy`) | Sélection d'une banque (`BankSelector`) et lancement du consentement (redirection vers l'URL d'autorisation de la banque) |
| `/banque/callback` | `RattacherBanqueCallback` | `RequireBudgy` (`budgy`) | Retour du consentement : finalisation, affichage des comptes rattachés ; gère aussi le cas renouvellement |
| `/comptes` | `MesComptes` | `RequireBudgy` (`budgy`) | Liste des comptes avec solde ; alertes de reconsentement et bannière de renouvellement ; rechargement sur événement du relais |
| `/comptes/:accountId` | `TransactionsCompte` | `RequireBudgy` (`budgy`) | Transactions paginées d'un compte, catégorisation et création de règle en un clic ; rechargement sur événement du relais |
| `/categories` | `Categories` | `RequireBudgy` (`budgy`) | Catégories du propriétaire : création (couleur + icône), modification, suppression |
| `/consentements` | `Consentements` | `RequireBudgy` (`budgy`) | Liste des consentements (statut, échéance) avec renouvellement des éligibles ; carte de rattachement d'une nouvelle banque |
| `*` | — | `RequireBudgy` (`budgy`) | Redirige vers `/` |

Les pages `/home`, `/transactions` et `/budgets` ont été retirées : l'accueil faisait doublon avec le tableau de bord, la liste transverse des transactions avec le détail d'un compte, et la saisie de budgets est devenue inutile depuis que le reste à dépenser se prédit seul.

### Tableau de bord

Trois blocs, **mois courant uniquement** (pas de navigation mensuelle) :

- **Soldes consolidés** — total tous comptes et détail par compte, avec le **solde à venir** quand la banque l'expose (Boursorama oui, Crédit Agricole non).
- **Reste à dépenser** — une ligne par catégorie, filtrables par un `Select` mono-catégorie, et un total égal à la somme des lignes. Le dépassement se lit au reste négatif.
- **Budget prévisionnel** — solde du mois et ses deux composantes (revenus, dépenses récurrentes), ou un état « données insuffisantes ».

Au montage du layout, le portail appelle la réconciliation côté API : les données se réparent seules, sans écran d'administration.

## Flux d'authentification

- La garde **`RequireBudgy`** (via `RouteGuard` de canopui) appelle `getMe` (`/auth/me`) et vérifie le rôle **`budgy`** (`isPortalBudgy`).
- **401** → redirection vers le portail Authenticator via `loginUrl()` (cookie `ch_redirect` + intention de redirection). Authentifié sans rôle `budgy` → `/forbidden`.
- Cookies de session (`ch_token` / `ch_refresh`, HttpOnly) same-origin via le proxy `/api`. La déconnexion (`/auth/logout`) renvoie au login.

> Le portail Budgy écoute sur **3203**, port qui ne figure pas dans la liste des origines de redirection de confiance du portail Authenticator (`VITE_TRUSTED_REDIRECT_ORIGINS` = 3201, 3202). Voir [Incohérences](#incohérences-relevées).

## API consommée

Client canopui `createApiClient({ basePath: "/api", withRefresh: true })` ; préfixes via la gateway : **`/api/auth`** (session) et **`/api/budgy`** (banques, consentements, comptes, transactions).

| Domaine | Endpoints (extraits) |
| --- | --- |
| Session | `/auth/me`, `/auth/logout` |
| Santé | `/budgy/health` |
| Banques | `/budgy/v1/banks` |
| Consentements | `/budgy/v1/consents` (GET/POST), `/budgy/v1/consents/callback` (POST), `/budgy/v1/consents/:id/renew` (POST) |
| Comptes | `/budgy/v1/accounts`, `/budgy/v1/balance` |
| Transactions | `/budgy/v1/accounts/:id/transactions?limit=&offset=`, `/budgy/v1/accounts/:id/transactions/:tx/category` (PUT), `/budgy/v1/accounts/:id/transactions/:tx/rule` (POST), `/budgy/v1/transactions/recategoriser` (POST) |
| Catégories | `/budgy/v1/categories` (GET/POST), `/budgy/v1/categories/:id` (PUT/DELETE) |
| Agrégats | `/budgy/v1/budgets/remaining?month=`, `/budgy/v1/forecast?month=` |

Voir la documentation de l'API : [../CH-Api-Budgy/README.md](../CH-Api-Budgy/README.md) et [../CH-Api-Authenticator/README.md](../CH-Api-Authenticator/README.md) — routées par la [gateway](../CH-Api-GateWay/README.md).

## Configuration (`.env`)

| Variable | Rôle | Exemple |
| --- | --- | --- |
| `PORT` | Port d'écoute du serveur (dev Vite / prod Express) | `3203` |
| `GATEWAY_URL` | URL de la gateway vers laquelle `/api` est proxifié | `http://localhost:8180` |
| `VITE_AUTH_PORTAL_URL` | URL du portail Authenticator (redirection si session absente/expirée) | `http://localhost:3200` |

Variables optionnelles du relais temps réel (lues par `src/relay/config.ts`, absentes de `.env.example` — le relais est désactivé si `VITE_RELAY_WS_URL` est vide) :

| Variable | Rôle |
| --- | --- |
| `VITE_RELAY_WS_URL` | URL WebSocket du broker MQTT (relais désactivé si absente) |
| `VITE_RELAY_TOPIC_PREFIX` | Préfixe des topics (défaut interne si absente) |
| `VITE_RELAY_TOKEN` | Jeton d'authentification du relais |

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

- **Documents légaux** : le dépôt fournit un `PRIVACY.md` (politique de confidentialité) et un `TERMS.md`, exposés dans l'interface via `LegalLinks` et `infoHref` (lien CGU vers le portail Authenticator). Budgy est décrit comme une application de gestion budgétaire à usage strictement personnel s'appuyant sur les services d'information sur les comptes (AIS) PSD2 via Enable Banking.
- **Flux de consentement PSD2** : sélection d'une banque → initiation d'un consentement → redirection vers l'URL d'autorisation de la banque → retour sur `/banque/callback` pour finalisation ; les consentements arrivant à échéance peuvent être renouvelés (même écran de callback réutilisé).
- **Relais temps réel (MQTT)** : optionnel ; lorsqu'il est configuré, les pages Comptes et Transactions se rechargent automatiquement sur événement, et un toast signale les erreurs de synchronisation.
- **Généralisation des banques** : les caisses régionales du Crédit Agricole sont fusionnées en une entrée « Crédit Agricole » (`lib/banks.ts`). L'API en expose une quarantaine, qui redirigent toutes vers la même page de sélection d'agence. Un Crédit Agricole hors France reste distinct.
- **Dépendance canopui non épinglée en pratique** : `package.json` déclare une borne (`^2.3.0`), mais le job `build` de la CI exécute `npm install canopui@latest` avant de builder — l'artefact déployé embarque toujours la dernière version publiée. Règle d'équipe : un composant manquant s'ajoute **au design system**, pas en local.

## Incohérences relevées

- Les variables `VITE_RELAY_*` du relais MQTT ne sont pas documentées dans `.env.example`.

> L'ancienne incohérence sur `VITE_TRUSTED_REDIRECT_ORIGINS` (Budgy absent des origines de confiance de l'Authenticator) est **résolue en production** : `https://ch-budgy.qvl-project.com` y figure désormais.
