<h1 align="center">📦 QVL · Private npm Registry</h1>

<p align="center">
  <em>Where QVL packages live, and how to install them from anywhere.</em><br>
  <em>Où vivent les paquets QVL, et comment les installer depuis n'importe où.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/registry-npm.qvl--project.com-cb3837?style=for-the-badge&logo=npm&logoColor=white" alt="Registry">
  <img src="https://img.shields.io/badge/engine-Verdaccio-4b5e40?style=for-the-badge" alt="Verdaccio">
  <img src="https://img.shields.io/badge/install-public%20·%20no%20account-3fb950?style=for-the-badge" alt="Public read">
  <img src="https://img.shields.io/badge/publish-authenticated%20only-6f42c1?style=for-the-badge" alt="Auth publish">
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

## 🎯 In one sentence

QVL packages (`canopui`, future `@qvl-*` packages) are published on a **private npm registry** (Verdaccio) hosted on the QVL machine and publicly reachable at **`https://npm.qvl-project.com/`** — anyone can **install** without an account; only authenticated users (and the CI) can **publish**.

## 🚨 The one thing everyone gets wrong — which URL from where?

The registry runs *on* the QVL machine, so two addresses exist for the same service. **They are not interchangeable:**

| You are… | Registry URL to use |
|---|---|
| 🌍 **Anywhere else** — a dev laptop, a CI job on another host, an AI coding agent, a teammate | ✅ **`https://npm.qvl-project.com/`** — the ONLY valid URL |
| 🖥️ On the QVL server itself (rare — infra maintenance) | `http://localhost:4873` also works |

> 🤖 **Note for AI agents & newcomers** — if you are working on any machine other than the QVL server, `localhost:4873` does **not** exist for you. It will time out or connect to something else entirely. Do not "try it first": go straight to `https://npm.qvl-project.com/`. No account, no token and no VPN are needed to install.

## 📥 Installing a QVL package (from anywhere)

```bash
# 1. at the root of the consuming project — one line, no secrets
echo "registry=https://npm.qvl-project.com/" > .npmrc

# 2. install
npm install canopui
```

That's all. Two useful properties:

- **No authentication needed to install** — QVL packages are readable by everyone (open source philosophy).
- **The registry proxies npmjs.org** — with that `.npmrc`, `react`, `@mui/material` and every public package still install normally (fetched through the registry, cached on the way).

## 📤 Publishing (maintainer & CI only)

Publishing is restricted to authenticated accounts. In practice **nobody publishes by hand**: pushing a git tag `vX.Y.Z` makes the <a href="../pipeline/ci-cd.md">CI pipeline</a> build, verify and publish the package automatically. The CI authenticates with a masked CI/CD variable — no credential lives in any repository.

## 🏗️ Under the hood

| Aspect | Detail |
|---|---|
| Engine | **Verdaccio** — lightweight private npm registry |
| Hosted | On the QVL machine, exposed through the <a href="tunnel-cloudflare.md">Cloudflare Tunnel</a> |
| Availability | Follows the machine — if it is off, installs fail until it is back (published versions are never lost) |
| Web UI | <a href="https://npm.qvl-project.com">npm.qvl-project.com</a> in a browser — browse packages & versions |
| Rights | install = everyone · publish/unpublish = authenticated only · account creation = open (`npm adduser`) |

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

## 🎯 En une phrase

Les paquets QVL (`canopui`, futurs paquets `@qvl-*`) sont publiés sur un **registre npm privé** (Verdaccio) hébergé sur la machine QVL et accessible publiquement sur **`https://npm.qvl-project.com/`** — tout le monde peut **installer** sans compte ; seuls les utilisateurs authentifiés (et la CI) peuvent **publier**.

## 🚨 LE piège classique — quelle URL depuis où ?

Le registre tourne *sur* la machine QVL : deux adresses existent donc pour le même service. **Elles ne sont pas interchangeables :**

| Tu es… | URL du registre à utiliser |
|---|---|
| 🌍 **Partout ailleurs** — un portable de dev, une CI sur une autre machine, un agent IA, un collègue | ✅ **`https://npm.qvl-project.com/`** — la SEULE URL valable |
| 🖥️ Sur le serveur QVL lui-même (rare — maintenance infra) | `http://localhost:4873` fonctionne aussi |

> 🤖 **Note pour les agents IA & nouveaux arrivants** — si tu travailles sur n'importe quelle machine autre que le serveur QVL, `localhost:4873` n'existe **pas** pour toi. Ça expirera en timeout, ou pire, ça touchera un tout autre service. Inutile « d'essayer d'abord » : va directement sur `https://npm.qvl-project.com/`. Aucun compte, aucun token, aucun VPN n'est nécessaire pour installer.

## 📥 Installer un paquet QVL (depuis n'importe où)

```bash
# 1. à la racine du projet consommateur — une ligne, aucun secret
echo "registry=https://npm.qvl-project.com/" > .npmrc

# 2. installer
npm install canopui
```

C'est tout. Deux propriétés utiles :

- **Aucune authentification pour installer** — les paquets QVL sont lisibles par tous (philosophie open source).
- **Le registre relaie npmjs.org** — avec ce `.npmrc`, `react`, `@mui/material` et tous les paquets publics s'installent normalement (récupérés à travers le registre, mis en cache au passage).

## 📤 Publier (mainteneur & CI uniquement)

La publication est réservée aux comptes authentifiés. En pratique **personne ne publie à la main** : pousser un tag git `vX.Y.Z` fait que la <a href="../pipeline/ci-cd.md">pipeline CI</a> construit, vérifie et publie le paquet automatiquement. La CI s'authentifie via une variable CI/CD masquée — aucun identifiant ne vit dans un dépôt.

## 🏗️ Sous le capot

| Aspect | Détail |
|---|---|
| Moteur | **Verdaccio** — registre npm privé léger |
| Hébergement | Sur la machine QVL, exposé via le <a href="tunnel-cloudflare.md">Cloudflare Tunnel</a> |
| Disponibilité | Suit la machine — éteinte, les installs échouent jusqu'à son retour (les versions publiées ne sont jamais perdues) |
| Interface web | <a href="https://npm.qvl-project.com">npm.qvl-project.com</a> dans un navigateur — parcourir paquets & versions |
| Droits | installer = tout le monde · publier/dépublier = authentifiés uniquement · création de compte = ouverte (`npm adduser`) |

</details>

---

<p align="center">
  <sub>© QVL — Documentation · <a href="../README.md">Hub</a> · Voir aussi : <a href="tunnel-cloudflare.md">Tunnel Cloudflare</a> · <a href="hebergement-gitlab.md">Hébergement GitLab</a> · <a href="../pipeline/ci-cd.md">CI/CD</a></sub><br>
  <sub>Crafted solo, with the help of agentic AI 🤖 · Conçu en solo, avec l'aide de l'IA agentique</sub>
</p>
