# HB-Front-StatBar

<p align="center">
  <em>StatBar SPA of the QVL Hobbies domain — rate, map and rank your favourite bars.</em><br>
  <em>SPA StatBar du domaine QVL Hobbies — noter, cartographier et classer ses bars préférés.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

<p align="center">
  <img alt="UI" src="https://img.shields.io/badge/React-19-61DAFB">
  <img alt="Bundler" src="https://img.shields.io/badge/Vite-8-646CFF">
  <img alt="Language" src="https://img.shields.io/badge/TypeScript-6-3178C6">
  <img alt="Design system" src="https://img.shields.io/badge/CanopUI-2.10.0-7C3AED">
  <img alt="Maps" src="https://img.shields.io/badge/Leaflet-OpenStreetMap-199900">
  <img alt="URL" src="https://img.shields.io/badge/statbar.qvl--project.com-live-brightgreen">
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

Single-page React application of the **Hobbies** domain: **StatBar**, the app to rate, map and rank bars. It consumes the **HB-Api-StatBar** service and is served **same-origin**, relaying `/api` to that API from its own web server.

## Features

- **Home** — presentation and quick access to the three sections (mobile-friendly layout).
- **Add a bar** — search a place by name (Photon / OpenStreetMap autocomplete, centred on Rouen) to auto-place the pin, or place it manually on the map; per-category rating via sliders.
- **Map** — all bars on an interactive Leaflet map with their scores.
- **Ranking** — sort by average or by rating category, with stars and a numeric rank.
- **Multilingual** — French (default) and English, hot-switch and remembered.
- **Polished UX** — CanopUI design system, organic animations, responsive.

## Stack

- **React 19** + **Vite 8** + TypeScript
- **CanopUI 2.10.0** — QVL-Studio design system (theme, components, i18n) on a **MUI 7** base
- **react-router-dom 7** — navigation
- **react-leaflet 5** + Leaflet + OpenStreetMap — maps
- **Node.js 20+** (ESM, Vite 8)

## Port & URL

- **Production** : `127.0.0.1:8312` (systemd unit `hb-statbar`), exposed behind Cloudflare Tunnel under **`statbar.qvl-project.com`**.
- **Dev** : Vite dev server on `http://localhost:5173`.

## Data access — same-origin `/api` relay

The front never hard-codes the API host. It calls a **relative `/api`** prefix; the production web server (and the Vite dev proxy) **relays `/api`** to **HB-Api-StatBar**, keeping API and front on the **same origin**.

| Variable | Dev default | Build value | Description |
|---|---|---|---|
| `VITE_API_URL` | `http://localhost:4000/api` | `/api` (relative) | API base URL (**includes** the `/api` prefix) |

The CI build injects `VITE_API_URL="/api"` so no absolute host is baked into the artifact — the bundle stays valid even if the domain changes.

## Deployment

Deployed via **`hb-deploy-node hb-statbar`** to **`/opt/hobbies/`** and run as a **systemd unit** under WSL. Only `dist/` is replaced on each deploy — the web server (`server.js`, `app.js`) and its dependencies live on the server and stay put. CI extends the shared `QVL-ToolBox/PipeLine` Node template (`lint` = ESLint, tests disabled, `build` with `VITE_API_URL=/api`) and runs on the group runner **`qvl-hobbies-wsl`**. The `update-checkout` job keeps the local clone under `/mnt/c/QVL/QVL-Hobbies/HB-Front-StatBar` in sync with `main`.

## Fonts

Typography is provided by CanopUI (**Chivo**, loaded via `canopui/styles.css`).

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

Application React monopage du domaine **Hobbies** : **StatBar**, l'app pour noter, cartographier et classer ses bars. Elle consomme le service **HB-Api-StatBar** et est servie **en même origine**, en relayant `/api` vers cette API depuis son propre serveur web.

## Fonctionnalités

- **Accueil** — présentation et accès rapide aux trois sections (layout adapté au mobile).
- **Ajouter un bar** — recherche du lieu par son nom (autocomplétion Photon / OpenStreetMap, centrée sur Rouen) qui place automatiquement le pin, ou placement manuel sur la carte ; notation par catégorie via sliders.
- **Carte** — tous les bars affichés sur une carte interactive Leaflet avec leurs notes.
- **Classement** — tri par moyenne ou par catégorie de note, avec étoiles et rang chiffré.
- **Multilingue** — Français (par défaut) et English, changement à chaud et mémorisé.
- **UX soignée** — design system CanopUI, animations organiques, responsive.

## Stack

- **React 19** + **Vite 8** + TypeScript
- **CanopUI 2.10.0** — design system QVL-Studio (thème, composants, i18n) sur socle **MUI 7**
- **react-router-dom 7** — navigation
- **react-leaflet 5** + Leaflet + OpenStreetMap — cartes
- **Node.js 20+** (ESM, Vite 8)

## Port & URL

- **Production** : `127.0.0.1:8312` (unité systemd `hb-statbar`), exposée derrière Cloudflare Tunnel sous **`statbar.qvl-project.com`**.
- **Dev** : serveur de dev Vite sur `http://localhost:5173`.

## Accès aux données — relais `/api` en même origine

Le front ne code jamais l'hôte de l'API en dur. Il appelle un préfixe **`/api` relatif** ; le serveur web de production (et le proxy Vite en dev) **relaie `/api`** vers **HB-Api-StatBar**, gardant l'API et le front sur la **même origine**.

| Variable | Défaut dev | Valeur de build | Description |
|---|---|---|---|
| `VITE_API_URL` | `http://localhost:4000/api` | `/api` (relatif) | URL de base de l'API (**inclut** le préfixe `/api`) |

Le build CI injecte `VITE_API_URL="/api"` : aucune URL absolue n'est figée dans l'artefact — le bundle reste valable même si le domaine change.

## Déploiement

Déployé via **`hb-deploy-node hb-statbar`** dans **`/opt/hobbies/`** et lancé en **unité systemd** sous WSL. Seul `dist/` est remplacé à chaque déploiement — le serveur web (`server.js`, `app.js`) et ses dépendances vivent sur le serveur et ne bougent pas. La CI étend le template Node partagé `QVL-ToolBox/PipeLine` (`lint` = ESLint, tests désactivés, `build` avec `VITE_API_URL=/api`) et tourne sur le runner de groupe **`qvl-hobbies-wsl`**. Le job `update-checkout` maintient le clone local sous `/mnt/c/QVL/QVL-Hobbies/HB-Front-StatBar` aligné sur `main`.

## Polices

La typographie est fournie par CanopUI (**Chivo**, chargée via `canopui/styles.css`).

</details>

---

<p align="center"><sub>© QVL — Documentation</sub></p>
