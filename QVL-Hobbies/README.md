# 🎮 QVL Hobbies

<p align="center">
  <em>Personal leisure & hobby apps — a static showcase fed solely by its author.</em><br>
  <em>Applications de loisirs personnelles — une vitrine statique alimentée par son seul auteur.</em>
</p>

<p align="center">
  <strong>🌐 Read in :</strong>&nbsp;
  <a href="#-english">🇬🇧 English</a>&nbsp;·&nbsp;
  <a href="#-français">🇫🇷 Français</a>
</p>

---

<!-- ============================== ENGLISH ============================== -->
<details open>
<summary><h3>🇬🇧&nbsp;&nbsp;English</h3></summary>

<a id="-english"></a>

## 📖 About this area

**QVL Hobbies** is the home of **personal leisure and hobby applications**. These apps are visitable by anyone, but they behave as a **static showcase for the general public** — a curated gallery that **only the author feeds and updates**.

This area documents the back-end services powering those hobby apps. Each service is a small, self-contained API; there is no shared gateway here, unlike CustHome.

## 🗄️ Documented repos

| Repo | Description | Documentation |
|---|---|---|
| HB-Api-Cocktail | Cocktails API (public read, local write) | [HB-Api-Cocktail/README.md](HB-Api-Cocktail/README.md) · [openapi.yaml](HB-Api-Cocktail/openapi.yaml) |

## 🔌 Default ports

| Service | Role | Port | Stack | Storage |
|---|---|---|---|---|
| HB-Api-Cocktail | Cocktail recipes, ingredient search, images | `127.0.0.1:8080` | Go (stdlib) | SQLite + FS |

## 🧱 Stack & conventions

- **Language** : Go (standard library HTTP, no framework).
- **Storage** : SQLite (pure-Go driver, no cgo) + local filesystem for images.
- **Contracts** : spec-first OpenAPI 3.0.3, embedded and served (`/openapi.yaml`, `/docs`).
- **Error format** : uniform JSON `{ "error": "...", "message": "..." }`.
- **Local-only writes** : write endpoints are reachable over loopback only and protected by a bearer token; read surfaces stay public.

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

## 📖 À propos de ce domaine

**QVL Hobbies** est le foyer des **applications de loisirs personnelles**. Ces applications sont visitables par tout le monde, mais fonctionnent comme une **vitrine statique pour le grand public** — une galerie soignée **que seul l'auteur alimente et met à jour**.

Ce domaine documente les services back-end qui alimentent ces applications de loisirs. Chaque service est une petite API autonome ; il n'y a pas de gateway partagée ici, contrairement à CustHome.

## 🗄️ Repos documentés

| Repo | Description | Documentation |
|---|---|---|
| HB-Api-Cocktail | API cocktails (lecture publique, écriture locale) | [HB-Api-Cocktail/README.md](HB-Api-Cocktail/README.md) · [openapi.yaml](HB-Api-Cocktail/openapi.yaml) |

## 🔌 Ports par défaut

| Service | Rôle | Port | Stack | Stockage |
|---|---|---|---|---|
| HB-Api-Cocktail | Recettes de cocktails, recherche par ingrédients, images | `127.0.0.1:8080` | Go (stdlib) | SQLite + FS |

## 🧱 Stack & conventions

- **Langage** : Go (HTTP via la bibliothèque standard, sans framework).
- **Stockage** : SQLite (driver pure-Go, sans cgo) + système de fichiers local pour les images.
- **Contrats** : OpenAPI 3.0.3 spec-first, embarqué et servi (`/openapi.yaml`, `/docs`).
- **Format d'erreur** : JSON uniforme `{ "error": "...", "message": "..." }`.
- **Écriture locale uniquement** : les endpoints d'écriture ne sont joignables qu'en loopback et protégés par un token bearer ; les surfaces de lecture restent publiques.

</details>
