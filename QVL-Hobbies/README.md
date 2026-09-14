# 🎮 QVL Hobbies

<p align="center">
  <em>Personal leisure & hobby apps — a public-facing showcase fed solely by its author.</em><br>
  <em>Applications de loisirs personnelles — une vitrine publique alimentée par son seul auteur.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

<p align="center">
  <img alt="Services" src="https://img.shields.io/badge/services-5-blue">
  <img alt="APIs" src="https://img.shields.io/badge/APIs-Go%20%C2%B7%20Node%2FMongo-00ADD8">
  <img alt="Fronts" src="https://img.shields.io/badge/fronts-React%20SPA%20%C3%97%203-61DAFB">
  <img alt="Runtime" src="https://img.shields.io/badge/runtime-systemd%20%2F%20WSL-333">
  <img alt="Runner" src="https://img.shields.io/badge/CI-qvl--hobbies--wsl-FC6D26">
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

## 📖 About this area

**QVL Hobbies** is the home of **personal leisure and hobby applications**. These apps are visitable by anyone, but they behave as a **public-facing showcase** — a curated gallery that **only the author feeds and updates**.

This area is a **mix of back-end APIs and front-end SPAs**, not a set of standalone Go APIs:

- **APIs** — one Go service (HB-Api-Cocktail, SQLite) and one Node/TypeScript service (HB-Api-StatBar, Express 5 + MongoDB).
- **Fronts** — three React + Vite single-page apps (StatBar, FondDeShaker, DéparteMental) built on the **CanopUI** design system.

There is **no shared gateway** here, unlike CustHome. Instead, each front that needs data is served **same-origin** and **relays `/api`** to its API from its own web server. DéparteMental is the exception: it is **100% static** with no backend at all.

## 🗄️ Documented repos

| Repo | Description | Documentation |
|---|---|---|
| HB-Api-Cocktail | Cocktails API — public read, local token write, public cookie preferences (Go + SQLite) | [HB-Api-Cocktail/README.md](HB-Api-Cocktail/README.md) · [openapi.yaml](HB-Api-Cocktail/openapi.yaml) |
| HB-Api-StatBar | Bar-rating API (Node + Express 5 + TypeScript + Mongoose/MongoDB) | [HB-Api-StatBar/README.md](HB-Api-StatBar/README.md) · [openapi.yaml](HB-Api-StatBar/openapi.yaml) |
| HB-Front-StatBar | StatBar SPA — rate, map & rank your bars (React + Vite + Leaflet + CanopUI) | [HB-Front-StatBar/README.md](HB-Front-StatBar/README.md) |
| HB-Front-FondDeShaker | FondDeShaker SPA — browse cocktails, "my bar" & favourites (React 19 + Vite + CanopUI) | [HB-Front-FondDeShaker/README.md](HB-Front-FondDeShaker/README.md) |
| HB-Front-DeparteMental | DéparteMental SPA — learn the 101 French *départements* (React 19 + Vite + CanopUI) | [HB-Front-DeparteMental/README.md](HB-Front-DeparteMental/README.md) |

> The GitLab project for the last one is **`qvl-hobbies/HB-DeparteMental`** (no `Front` segment), even though its local clone folder is `HB-Front-DeparteMental`.

## 🔌 Default ports

Production ports are assigned in the **`831x`** range. `8080` is **never** used in production — it is only a code fallback for HB-Api-Cocktail.

| Service | Role | Prod port | Public URL | Stack | Storage |
|---|---|---|---|---|---|
| HB-Api-Cocktail | Cocktail recipes, ingredient search, images, preferences | `127.0.0.1:8310` | api-cocktail.qvl-project.com | Go (stdlib) | SQLite + FS |
| HB-Api-StatBar | Bar rating (CRUD) | `127.0.0.1:8311` | *(internal, via HB-Front-StatBar `/api`)* | Node + Express 5 + TS | MongoDB |
| HB-Front-StatBar | StatBar SPA, relays `/api` → HB-Api-StatBar | `127.0.0.1:8312` | statbar.qvl-project.com | React + Vite + TS + Leaflet | — (static build) |
| HB-Front-FondDeShaker | FondDeShaker SPA, relays `/api` → HB-Api-Cocktail (`/api` prefix stripped) | `127.0.0.1:8313` | fonddeshaker.qvl-project.com | React 19 + Vite + TS | — (static build) |
| HB-Front-DeparteMental | DéparteMental SPA, 100% static (progress in `localStorage`, **no backend, no relay**) | `127.0.0.1:8314` | departemental.qvl-project.com | React 19 + Vite + TS | — (`localStorage`) |

> `HB-Api-StatBar` listens on **8311** in production; its **in-code default is `4000`** (`PORT`). Fronts are served by a small Node web server that also relays `/api` to keep the API **same-origin** — a hard requirement for the `HttpOnly` preference cookies used by FondDeShaker (Safari/ITP blocks third-party cookies).

## 🧱 Stack & conventions

The domain has **no single stack**: each service brings its own. The section below documents **HB-Api-Cocktail only**; the other services follow their own conventions (see their READMEs).

- **HB-Api-Cocktail** — Go (standard-library HTTP, no framework); SQLite (pure-Go `modernc.org/sqlite`, no cgo) + local filesystem for images; spec-first **OpenAPI 3.0.3**, embedded and served (`/openapi.yaml`, `/docs`); uniform JSON error format `{ "error": "...", "message": "..." }`; **`POST /cocktails`** is loopback + bearer token only, while `GET`/`PUT /preferences` are **public** (cookie-backed, stateless) and every read surface stays public.
- **HB-Api-StatBar** — Node + **Express 5** + TypeScript + **Mongoose/MongoDB**; REST CRUD under `/api/bars`; no OpenAPI embedding (endpoints documented from the routes).
- **Fronts (StatBar, FondDeShaker, DéparteMental)** — React + Vite + TypeScript on the **CanopUI** design system (private registry `npm.qvl-project.com`); the two data-driven fronts call a **relative `/api`** prefix and rely on their web server relaying it same-origin; DéparteMental is fully static and keeps its progress in `localStorage`.

## 🚀 Deployment & conventions

- **Runtime** — every service runs as a **systemd unit under WSL**, installed in **`/opt/hobbies/`**.
- **Deploy helpers** — fronts and the Node API are deployed via **`hb-deploy-node`** (units `hb-statbar`, `hb-fonddeshaker`, `hb-departemental`, `hb-api-statbar`); the Go API is deployed via **`ch-deploy-bin`** (unit `hb-api-cocktail`).
- **CI/CD** — a dedicated **GitLab group runner `qvl-hobbies-wsl`** runs the pipelines (**shared runners disabled** for the group). Pipelines extend the shared templates from **`QVL-ToolBox/PipeLine`** (`go.gitlab-ci.yml` / `node.gitlab-ci.yml`).
- **`update-checkout` job** — after each deploy on `main`, an auto-pull job fast-forwards the persistent local clone under **`/mnt/c/QVL/QVL-Hobbies/*`**, so the working copies never drift from `main`.
- **URL convention** — **no `hb-` prefix in public URLs** (`statbar.qvl-project.com`, not `hb-statbar…`). The `hb-` prefix is kept **only** for repository names and systemd unit names.

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

## 📖 À propos de ce domaine

**QVL Hobbies** est le foyer des **applications de loisirs personnelles**. Ces applications sont visitables par tout le monde, mais fonctionnent comme une **vitrine publique** — une galerie soignée **que seul l'auteur alimente et met à jour**.

Ce domaine est un **mélange d'APIs back-end et de fronts SPA**, et non un ensemble de petites API Go autonomes :

- **APIs** — un service Go (HB-Api-Cocktail, SQLite) et un service Node/TypeScript (HB-Api-StatBar, Express 5 + MongoDB).
- **Fronts** — trois SPA React + Vite (StatBar, FondDeShaker, DéparteMental) bâties sur le design system **CanopUI**.

Il n'y a **pas de gateway partagée** ici, contrairement à CustHome. À la place, chaque front qui a besoin de données est servi **en même origine** et **relaie `/api`** vers son API depuis son propre serveur web. DéparteMental fait exception : il est **100 % statique**, sans aucun backend.

## 🗄️ Repos documentés

| Repo | Description | Documentation |
|---|---|---|
| HB-Api-Cocktail | API cocktails — lecture publique, écriture locale par token, préférences publiques par cookies (Go + SQLite) | [HB-Api-Cocktail/README.md](HB-Api-Cocktail/README.md) · [openapi.yaml](HB-Api-Cocktail/openapi.yaml) |
| HB-Api-StatBar | API de notation de bars (Node + Express 5 + TypeScript + Mongoose/MongoDB) | [HB-Api-StatBar/README.md](HB-Api-StatBar/README.md) · [openapi.yaml](HB-Api-StatBar/openapi.yaml) |
| HB-Front-StatBar | SPA StatBar — noter, cartographier & classer ses bars (React + Vite + Leaflet + CanopUI) | [HB-Front-StatBar/README.md](HB-Front-StatBar/README.md) |
| HB-Front-FondDeShaker | SPA FondDeShaker — parcourir les cocktails, « mon bar » & favoris (React 19 + Vite + CanopUI) | [HB-Front-FondDeShaker/README.md](HB-Front-FondDeShaker/README.md) |
| HB-Front-DeparteMental | SPA DéparteMental — apprendre les 101 départements français (React 19 + Vite + CanopUI) | [HB-Front-DeparteMental/README.md](HB-Front-DeparteMental/README.md) |

> Le projet GitLab du dernier est **`qvl-hobbies/HB-DeparteMental`** (sans segment `Front`), bien que le dossier du clone local soit `HB-Front-DeparteMental`.

## 🔌 Ports par défaut

Les ports de production sont attribués dans la plage **`831x`**. Le port `8080` n'est **jamais** utilisé en production — c'est seulement une valeur de repli du code de HB-Api-Cocktail.

| Service | Rôle | Port prod | URL publique | Stack | Stockage |
|---|---|---|---|---|---|
| HB-Api-Cocktail | Recettes de cocktails, recherche par ingrédients, images, préférences | `127.0.0.1:8310` | api-cocktail.qvl-project.com | Go (stdlib) | SQLite + FS |
| HB-Api-StatBar | Notation de bars (CRUD) | `127.0.0.1:8311` | *(interne, via HB-Front-StatBar `/api`)* | Node + Express 5 + TS | MongoDB |
| HB-Front-StatBar | SPA StatBar, relaie `/api` → HB-Api-StatBar | `127.0.0.1:8312` | statbar.qvl-project.com | React + Vite + TS + Leaflet | — (build statique) |
| HB-Front-FondDeShaker | SPA FondDeShaker, relaie `/api` → HB-Api-Cocktail (préfixe `/api` retiré) | `127.0.0.1:8313` | fonddeshaker.qvl-project.com | React 19 + Vite + TS | — (build statique) |
| HB-Front-DeparteMental | SPA DéparteMental, 100 % statique (progression en `localStorage`, **aucun backend, aucun relais**) | `127.0.0.1:8314` | departemental.qvl-project.com | React 19 + Vite + TS | — (`localStorage`) |

> `HB-Api-StatBar` écoute sur **8311** en production ; sa **valeur par défaut dans le code est `4000`** (`PORT`). Les fronts sont servis par un petit serveur web Node qui relaie aussi `/api` afin de garder l'API **en même origine** — une exigence dure pour les cookies de préférences `HttpOnly` utilisés par FondDeShaker (Safari/ITP bloque les cookies tiers).

## 🧱 Stack & conventions

Le domaine n'a **pas de stack unique** : chaque service apporte la sienne. La section ci-dessous décrit **HB-Api-Cocktail uniquement** ; les autres services suivent leurs propres conventions (voir leurs README).

- **HB-Api-Cocktail** — Go (HTTP via la bibliothèque standard, sans framework) ; SQLite (driver pure-Go `modernc.org/sqlite`, sans cgo) + système de fichiers local pour les images ; **OpenAPI 3.0.3** spec-first, embarqué et servi (`/openapi.yaml`, `/docs`) ; format d'erreur JSON uniforme `{ "error": "...", "message": "..." }` ; **`POST /cocktails`** en loopback + token bearer uniquement, tandis que `GET`/`PUT /preferences` sont **publics** (portés par cookies, sans état) et toutes les surfaces de lecture restent publiques.
- **HB-Api-StatBar** — Node + **Express 5** + TypeScript + **Mongoose/MongoDB** ; CRUD REST sous `/api/bars` ; pas d'OpenAPI embarqué (endpoints documentés d'après les routes).
- **Fronts (StatBar, FondDeShaker, DéparteMental)** — React + Vite + TypeScript sur le design system **CanopUI** (registre privé `npm.qvl-project.com`) ; les deux fronts pilotés par données appellent un préfixe **`/api` relatif** et comptent sur leur serveur web pour le relayer en même origine ; DéparteMental est entièrement statique et conserve sa progression en `localStorage`.

## 🚀 Déploiement & conventions

- **Runtime** — chaque service tourne en **unité systemd sous WSL**, installée dans **`/opt/hobbies/`**.
- **Helpers de déploiement** — les fronts et l'API Node sont déployés via **`hb-deploy-node`** (unités `hb-statbar`, `hb-fonddeshaker`, `hb-departemental`, `hb-api-statbar`) ; l'API Go est déployée via **`ch-deploy-bin`** (unité `hb-api-cocktail`).
- **CI/CD** — un **runner de groupe GitLab dédié `qvl-hobbies-wsl`** exécute les pipelines (**shared runners désactivés** pour le groupe). Les pipelines étendent les templates partagés de **`QVL-ToolBox/PipeLine`** (`go.gitlab-ci.yml` / `node.gitlab-ci.yml`).
- **Job `update-checkout`** — après chaque déploiement sur `main`, un job d'auto-pull avance en fast-forward le clone local persistant sous **`/mnt/c/QVL/QVL-Hobbies/*`**, pour que les copies de travail ne dérivent jamais de `main`.
- **Convention d'URL** — **plus de préfixe `hb-` dans les URL publiques** (`statbar.qvl-project.com`, pas `hb-statbar…`). Le préfixe `hb-` est gardé **uniquement** pour les noms de dépôts et d'unités systemd.

</details>

---

<p align="center"><sub>© QVL — Documentation</sub></p>
