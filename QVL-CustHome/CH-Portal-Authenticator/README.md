# CH-Portal-Authenticator

Portail d'authentification de l'écosystème **CustHome** : point d'entrée unique (SSO) où les utilisateurs se connectent, s'inscrivent, réinitialisent leur mot de passe et gèrent leur compte. Tous les autres portails (Admin, Drive, Budgy) y renvoient l'utilisateur non authentifié.

![React](https://img.shields.io/badge/React-19-61dafb) ![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6) ![Vite](https://img.shields.io/badge/Vite-7-646cff) ![canopui](https://img.shields.io/badge/canopui-1.0.1-lightgrey) ![Express](https://img.shields.io/badge/Express-5-000000)

## Stack

- **React 19** + **TypeScript** (strict), routage via **react-router-dom 7**
- **canopui** (UI library QVL) — thème, i18n, composants (`Form`, `InputEmail`, `Feedback`, `Spinner`…)
- **MUI 7** + Emotion (socle de canopui)
- Build **Vite 7**, tests **Vitest**
- Serveur de production **Express** (`server.js` + `app.js`) qui sert le build et proxifie `/api`

## Port par défaut

| Contexte | Port |
| --- | --- |
| Dev (Vite, `.env` `PORT`) | **3200** |
| Prod (`server.js`, fallback `PORT`) | 3000 |

> Le port documenté de référence est **3200** (`.env.example`, `vite.config.ts`). Voir la section [Incohérences](#incohérences-relevées).

## Routes / pages

Toutes les routes sont **publiques** (aucune garde de rôle) et rendues dans un `Layout` commun. `App.tsx` redirige `/` et toute route inconnue vers `/login`.

| Chemin | Page | Accès | Description |
| --- | --- | --- | --- |
| `/` | — | Public | Redirige vers `/login` |
| `/login` | `Login` | Public | Formulaire e-mail + mot de passe ; liens vers mot de passe oublié et inscription |
| `/register` | `Register` | Public | Inscription (nom, e-mail, mot de passe + confirmation, acceptation des CGU). Masqué si l'inscription est désactivée côté serveur (`/settings/registration`) |
| `/forgot-password` | `ForgotPassword` | Public | Demande d'e-mail de réinitialisation |
| `/reset-password` | `ResetPassword` | Public | Saisie d'un nouveau mot de passe à partir d'un token |
| `/pending` | `PendingValidation` | Public | Message d'attente : compte en attente de validation par un administrateur |
| `/cgu` | `Cgu` | Public | Conditions générales d'utilisation + mentions légales (défilement vers l'ancre via le hash) |
| `/account` | `Account` | Public* | Détails du profil connecté (`/me`) et bouton de déconnexion |
| `*` | — | Public | Redirige vers `/login` |

\* `/account` ne possède pas de garde de route dédiée : la protection repose sur l'API (`/auth/me` renvoie 401 si la session est absente).

## Flux d'authentification

Ce portail est **l'émetteur** du SSO CustHome ; les autres portails en sont consommateurs.

1. Un portail consommateur (Admin/Drive/Budgy) sans session valide dépose un cookie `ch_redirect` (URL de retour) puis redirige vers `.../login?<intention de redirection>`.
2. L'utilisateur se connecte ici. La gateway pose les cookies **HttpOnly** de session (`ch_token` / `ch_refresh`).
3. Après connexion, le portail lit le cookie `ch_redirect` **uniquement si** l'intention de redirection est présente, puis applique `safeRedirect` :
   - chemin interne (`/...`) → autorisé ;
   - URL absolue → autorisée **seulement** si son origine figure dans `VITE_TRUSTED_REDIRECT_ORIGINS` (anti open-redirect) ;
   - sinon repli sur `/account`.
4. Le navigateur reste **same-origin** : le serveur Express proxifie `/api` vers la gateway, ce qui permet aux cookies HttpOnly d'être transmis.

Le domaine du cookie de retour est vide en local (localhost partagé) et positionné en production pour être lisible par les sous-domaines des autres portails.

## API consommée

Appels via la gateway, préfixe **`/api/auth`** (client `createApiClient({ basePath: "/api/auth" })`).

| Fonction | Endpoint | Méthode |
| --- | --- | --- |
| `login` | `/login` | POST |
| `register` | `/register` | POST |
| `forgotPassword` | `/password/forgot` | POST |
| `resetPassword` | `/password/reset` | POST |
| `getRegistrationEnabled` | `/settings/registration` | GET |
| `getMe` | `/me` | GET |
| `logout` | `/logout` | POST |

Voir la documentation de l'API : [../CH-Api-Authenticator/README.md](../CH-Api-Authenticator/README.md) — routée par la [gateway](../CH-Api-GateWay/README.md).

## Configuration (`.env`)

| Variable | Rôle | Exemple |
| --- | --- | --- |
| `PORT` | Port d'écoute du serveur (dev Vite / prod Express) | `3200` |
| `GATEWAY_URL` | URL de la gateway vers laquelle `/api` est proxifié | `http://localhost:8180` |
| `VITE_TRUSTED_REDIRECT_ORIGINS` | Origines de portails autorisées en redirection post-login (CSV, anti open-redirect) | `http://localhost:3201,http://localhost:3202,http://[::1]:3201,http://[::1]:3202` |
| `VITE_REDIRECT_COOKIE_DOMAIN` | Domaine du cookie de retour SSO (vide en local) | `.qvl-project.com` |

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

- **CGU** : page `/cgu` dédiée (composants `CguTerms` + `LegalNotice`), lien obligatoire à l'inscription via une case à cocher renvoyant la version des termes acceptée (`accepted_terms_version`).
- **Inscription conditionnelle** : le formulaire `Register` interroge `/settings/registration` ; si l'inscription est désactivée, il affiche un message d'information au lieu du formulaire.
- **Portail émetteur du SSO** : c'est le seul des quatre portails sans garde de rôle ; toutes ses pages sont publiques.

## Incohérences relevées

- Le fallback de port dans `server.js` est **3000**, alors que `.env.example` et `vite.config.ts` utilisent **3200**. En pratique `PORT` est fourni via l'environnement.
- Le fallback `GATEWAY_URL` dans `server.js` est `http://localhost:8080`, alors que `.env.example` et `vite.config.ts` utilisent `http://localhost:8180`.
- `VITE_TRUSTED_REDIRECT_ORIGINS` (et son défaut dans `redirect.ts`) liste uniquement **3201 (Admin)** et **3202 (Drive)** : le portail **Budgy (3203)** n'y figure pas, une redirection post-login vers Budgy retomberait donc sur `/account`.
- `package.json` déclare une dépendance `ch-portail-admin` (`file:../CH-Portail-Admin`) inhabituelle pour un portail d'authentification.
