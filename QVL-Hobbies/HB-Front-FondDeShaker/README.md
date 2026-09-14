# HB-Front-FondDeShaker

<p align="center">
  <em>FondDeShaker SPA of the QVL Hobbies domain — browse cocktails, "my bar" & favourites.</em><br>
  <em>SPA FondDeShaker du domaine QVL Hobbies — parcourir les cocktails, « mon bar » & favoris.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

<p align="center">
  <img alt="UI" src="https://img.shields.io/badge/React-19-61DAFB">
  <img alt="Bundler" src="https://img.shields.io/badge/Vite-7-646CFF">
  <img alt="Language" src="https://img.shields.io/badge/TypeScript-strict-3178C6">
  <img alt="Design system" src="https://img.shields.io/badge/CanopUI-%5E2.11.0-7C3AED">
  <img alt="Router" src="https://img.shields.io/badge/react--router-7-CA4245">
  <img alt="URL" src="https://img.shields.io/badge/fonddeshaker.qvl--project.com-live-brightgreen">
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

Single-page React application of the **Hobbies** domain: **FondDeShaker**, which lets you search cocktails by name, by ingredients or by the contents of "my bar", and manage a favourites list. It consumes the **HB-Api-Cocktail** service (Go + SQLite) and is served **same-origin**, relaying `/api` to that API.

The **UI is entirely in French** (CanopUI i18n, single locale); the **content served by the API stays in English**, deliberately untranslated.

## Features

- **My bar** (`/bar`) — pick the ingredients you own.
- **Search** (`/recherche`, the home route) — by name, by ingredients, or "makeable from my bar".
- **Favourites** (`/favoris`) — a curated list.
- **Recipe overlay** — a shareable detail view driven by the `?cocktail=<id>` URL param (not a route), with browser-back closing it.

## Stack

- **React 19** + **Vite 7** + TypeScript (strict)
- **CanopUI ^2.11.0** — QVL-Studio design system on a **MUI 7** base (private registry `npm.qvl-project.com`)
- **motion ^12** — declared explicitly (it is a *dependency* of CanopUI, not a peer)
- **react-router 7** — navigation and the recipe overlay
- No state manager, no React Query, no cookie library: a hand-rolled typed `fetch` with per-field response validation, and `useSyncExternalStore` stores for the catalogue and preferences.

## Port & URL

- **Production** : `127.0.0.1:8313` (systemd unit `hb-fonddeshaker`), exposed behind Cloudflare Tunnel under **`fonddeshaker.qvl-project.com`**.
- **Dev** : Vite dev server on `http://127.0.0.1:5173` (`strictPort`, host pinned to `127.0.0.1`).

## Data access — same-origin `/api` relay (prefix stripped)

The front **always** calls a **relative `/api`** prefix. The production web server (and the Vite dev proxy) relays `/api` to **HB-Api-Cocktail**, **stripping the `/api` prefix** before forwarding — the API exposes its routes at the root, so `/api/health` becomes `/health` on the API side.

| Variable | Default | Description |
|---|---|---|
| `VITE_API_BASE_URL` | `http://127.0.0.1:8090` | **Dev proxy target only** — the origin where the API listens in development. Never used as a browser `fetch` base (that would make calls cross-origin and break cookies). |

**Preferences (bar & favourites)** live in two **`HttpOnly` cookies** (`hb_bar`, `hb_fav`) set and read by the API via `GET`/`PUT /preferences`; the front never touches them in JavaScript. This is why the API **must be served under the same origin** as the front in production — a distinct domain would turn the cookie into a third-party cookie that Safari/ITP blocks outright.

## Deployment

Deployed via **`hb-deploy-node hb-fonddeshaker`** to **`/opt/hobbies/`** and run as a **systemd unit** under WSL; only `dist/` is replaced on each deploy. The app calls `/api` in hard, so nothing is injected at build time — the server relays to HB-Api-Cocktail. CI extends the shared `QVL-ToolBox/PipeLine` Node template with **all quality jobs enabled** (ESLint, Prettier `format:check`, Vitest, build) and runs on the group runner **`qvl-hobbies-wsl`**. The `update-checkout` job keeps the local clone under `/mnt/c/QVL/QVL-Hobbies/HB-Front-FondDeShaker` in sync with `main`.

## Theme & i18n

- Theme (light/dark) persisted under the app-specific key **`fonddeshaker-theme`** (never the shared `ch-theme-mode`).
- Single UI locale (`fr`); `localStorage` is used **only** for the theme, never for user data.

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

Application React monopage du domaine **Hobbies** : **FondDeShaker**, qui permet de rechercher des cocktails par nom, par ingrédients ou selon le contenu de « mon bar », et de gérer une liste de favoris. Elle consomme le service **HB-Api-Cocktail** (Go + SQLite) et est servie **en même origine**, en relayant `/api` vers cette API.

L'**UI est entièrement en français** (i18n CanopUI, locale unique) ; le **contenu servi par l'API reste en anglais**, assumé et non traduit.

## Fonctionnalités

- **Mon bar** (`/bar`) — sélectionner les ingrédients possédés.
- **Recherche** (`/recherche`, route d'accueil) — par nom, par ingrédients, ou « réalisable depuis mon bar ».
- **Favoris** (`/favoris`) — une liste soignée.
- **Overlay de recette** — vue de détail partageable pilotée par le paramètre d'URL `?cocktail=<id>` (pas une route), que le bouton retour du navigateur ferme.

## Stack

- **React 19** + **Vite 7** + TypeScript (strict)
- **CanopUI ^2.11.0** — design system QVL-Studio sur socle **MUI 7** (registre privé `npm.qvl-project.com`)
- **motion ^12** — déclaré explicitement (c'est une *dependency* de CanopUI, pas une peer)
- **react-router 7** — navigation et overlay de recette
- Pas de state manager, pas de React Query, pas de lib de cookies : un `fetch` typé maison avec validation champ par champ des réponses, et des stores `useSyncExternalStore` pour le catalogue et les préférences.

## Port & URL

- **Production** : `127.0.0.1:8313` (unité systemd `hb-fonddeshaker`), exposée derrière Cloudflare Tunnel sous **`fonddeshaker.qvl-project.com`**.
- **Dev** : serveur de dev Vite sur `http://127.0.0.1:5173` (`strictPort`, hôte fixé à `127.0.0.1`).

## Accès aux données — relais `/api` en même origine (préfixe retiré)

Le front appelle **toujours** un préfixe **`/api` relatif**. Le serveur web de production (et le proxy Vite en dev) relaie `/api` vers **HB-Api-Cocktail** en **retirant le préfixe `/api`** avant transmission — l'API expose ses routes à la racine, donc `/api/health` devient `/health` côté API.

| Variable | Défaut | Description |
|---|---|---|
| `VITE_API_BASE_URL` | `http://127.0.0.1:8090` | **Cible du proxy de dev uniquement** — l'origine où écoute l'API en développement. Jamais utilisée comme base de `fetch` côté navigateur (cela rendrait les appels cross-origin et casserait les cookies). |

**Les préférences (bar & favoris)** vivent dans deux **cookies `HttpOnly`** (`hb_bar`, `hb_fav`) posés et relus par l'API via `GET`/`PUT /preferences` ; le front n'y touche jamais en JavaScript. C'est pourquoi l'API **doit être servie sous la même origine** que le front en production — un domaine distinct ferait du cookie un cookie tiers que Safari/ITP bloque purement et simplement.

## Déploiement

Déployé via **`hb-deploy-node hb-fonddeshaker`** dans **`/opt/hobbies/`** et lancé en **unité systemd** sous WSL ; seul `dist/` est remplacé à chaque déploiement. L'application appelle `/api` en dur, donc rien n'est injecté au build — le serveur relaie vers HB-Api-Cocktail. La CI étend le template Node partagé `QVL-ToolBox/PipeLine` avec **tous les jobs de qualité activés** (ESLint, Prettier `format:check`, Vitest, build) et tourne sur le runner de groupe **`qvl-hobbies-wsl`**. Le job `update-checkout` maintient le clone local sous `/mnt/c/QVL/QVL-Hobbies/HB-Front-FondDeShaker` aligné sur `main`.

## Thème & i18n

- Thème (clair/sombre) persisté sous la clé propre à l'application **`fonddeshaker-theme`** (jamais la clé partagée `ch-theme-mode`).
- Locale d'UI unique (`fr`) ; `localStorage` n'est utilisé **que** pour le thème, jamais pour des données utilisateur.

</details>

---

<p align="center"><sub>© QVL — Documentation</sub></p>
