# HB-Front-DeparteMental

<p align="center">
  <em>DéparteMental SPA of the QVL Hobbies domain — learn the 101 French départements. Fully static, no backend.</em><br>
  <em>SPA DéparteMental du domaine QVL Hobbies — apprendre les 101 départements français. 100% statique, aucun backend.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

<p align="center">
  <img alt="UI" src="https://img.shields.io/badge/React-19-61DAFB">
  <img alt="Bundler" src="https://img.shields.io/badge/Vite-7-646CFF">
  <img alt="Language" src="https://img.shields.io/badge/TypeScript-5-3178C6">
  <img alt="Design system" src="https://img.shields.io/badge/CanopUI-3.0.1-7C3AED">
  <img alt="Backend" src="https://img.shields.io/badge/backend-none%20(static)-lightgrey">
  <img alt="URL" src="https://img.shields.io/badge/departemental.qvl--project.com-live-brightgreen">
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

Single-page React application of the **Hobbies** domain: **DéparteMental**, a game to finally memorise the 101 French *départements* — codes, names, prefectures and sub-prefectures. It is **100% static**: **no account, no backend, no `/api` relay**; all progress is kept in the browser's **`localStorage`**.

> The GitLab project is **`qvl-hobbies/HB-DeparteMental`** (no `Front` segment), even though the local clone folder is `HB-Front-DeparteMental`.

## Game modes

- ⚡ **Quiz éclair** — 60 seconds, MCQ or keyboard input, streak with a multiplier.
- 📚 **Entraînement** — one theme of your choice (prefectures, sub-prefectures, codes, names, regions), 10 lives, questions drawn towards your weak spots.
- 📅 **Défi du jour** — one mystery *département* per day, progressive hints, shareable result.
- 🗺️ **Carte** — locate *départements* on the map of France + a progress heatmap.

## Stack

- **React 19** + **Vite 7** + TypeScript
- **CanopUI 3.0.1** (pinned exact) — QVL-Studio design system providing the theme, components (`Card`, `Choice`, `SvgMap`, `Lives`…), sounds and animated background (private registry `npm.qvl-project.com`)
- **@mui/material ^9**, **framer-motion ^13** — bounds set by CanopUI's peer dependencies
- **@svg-maps/france.departments** — the interactive map
- Data source: `départements.csv.txt` → `src/data/departements.json` (`npm run data`)

## Port & URL

- **Production** : `127.0.0.1:8314` (systemd unit `hb-departemental`), exposed behind Cloudflare Tunnel under **`departemental.qvl-project.com`**.
- **Dev** : Vite dev server on `http://localhost:5180`.

## No backend

There is **no API, no relay and no environment variable to inject at build**. Everything runs client-side; progress persists in `localStorage`. The `canopyVideo()` Vite plugin (shipped with CanopUI) serves the animated background scenes as files (dev middleware, copied into the output at build) instead of inlining them in the bundle.

## Deployment

Deployed via **`hb-deploy-node hb-departemental`** to **`/opt/hobbies/`** and run as a **systemd unit** under WSL; only `dist/` is replaced on each deploy. CI extends the shared `QVL-ToolBox/PipeLine` Node template (`lint` = ESLint, tests disabled, `build` = `tsc -b && vite build`) and runs on the group runner **`qvl-hobbies-wsl`**. The `update-checkout` job keeps the local clone under `/mnt/c/QVL/QVL-Hobbies/HB-Front-DeparteMental` in sync with `main`.

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

Application React monopage du domaine **Hobbies** : **DéparteMental**, un jeu pour enfin retenir les 101 départements français — codes, noms, préfectures et sous-préfectures. Elle est **100 % statique** : **pas de compte, pas de backend, pas de relais `/api`** ; toute la progression est conservée dans le **`localStorage`** du navigateur.

> Le projet GitLab est **`qvl-hobbies/HB-DeparteMental`** (sans segment `Front`), bien que le dossier du clone local soit `HB-Front-DeparteMental`.

## Modes de jeu

- ⚡ **Quiz éclair** — 60 secondes, QCM ou saisie clavier, streak avec multiplicateur.
- 📚 **Entraînement** — un thème au choix (préfectures, sous-préfectures, codes, noms, régions), 10 vies, questions à la chaîne tirées vers tes points faibles.
- 📅 **Défi du jour** — un département mystère par jour, indices progressifs, résultat partageable.
- 🗺️ **Carte** — localiser les départements sur la carte de France + heatmap de progression.

## Stack

- **React 19** + **Vite 7** + TypeScript
- **CanopUI 3.0.1** (épinglée à l'exact) — design system QVL-Studio fournissant le thème, les composants (`Card`, `Choice`, `SvgMap`, `Lives`…), les sons et le fond animé (registre privé `npm.qvl-project.com`)
- **@mui/material ^9**, **framer-motion ^13** — bornes fixées par les dépendances de pair de CanopUI
- **@svg-maps/france.departments** — la carte interactive
- Source de données : `départements.csv.txt` → `src/data/departements.json` (`npm run data`)

## Port & URL

- **Production** : `127.0.0.1:8314` (unité systemd `hb-departemental`), exposée derrière Cloudflare Tunnel sous **`departemental.qvl-project.com`**.
- **Dev** : serveur de dev Vite sur `http://localhost:5180`.

## Aucun backend

Il n'y a **ni API, ni relais, ni variable d'environnement à injecter au build**. Tout tourne côté client ; la progression persiste en `localStorage`. Le plugin Vite `canopyVideo()` (livré avec CanopUI) sert les scènes du fond animé comme des fichiers (middleware en dev, recopie dans la sortie au build) au lieu de les inliner dans le bundle.

## Déploiement

Déployé via **`hb-deploy-node hb-departemental`** dans **`/opt/hobbies/`** et lancé en **unité systemd** sous WSL ; seul `dist/` est remplacé à chaque déploiement. La CI étend le template Node partagé `QVL-ToolBox/PipeLine` (`lint` = ESLint, tests désactivés, `build` = `tsc -b && vite build`) et tourne sur le runner de groupe **`qvl-hobbies-wsl`**. Le job `update-checkout` maintient le clone local sous `/mnt/c/QVL/QVL-Hobbies/HB-Front-DeparteMental` aligné sur `main`.

</details>

---

<p align="center"><sub>© QVL — Documentation</sub></p>
