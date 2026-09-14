<h1 align="center">☁️ QVL · Cloudflare Tunnel</h1>

<p align="center">
  <em>One home machine, many public websites — with zero open ports.</em><br>
  <em>Une machine à la maison, plusieurs sites publics — sans aucun port ouvert.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/domain-qvl--project.com-f38020?style=for-the-badge&logo=cloudflare&logoColor=white" alt="Domain">
  <img src="https://img.shields.io/badge/tunnel-qvl--infra-f38020?style=for-the-badge&logo=cloudflare&logoColor=white" alt="Tunnel">
  <img src="https://img.shields.io/badge/HTTPS-automatic-3fb950?style=for-the-badge" alt="HTTPS">
  <img src="https://img.shields.io/badge/open%20router%20ports-zero-6f42c1?style=for-the-badge" alt="No open ports">
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

A **Cloudflare Tunnel** makes services hosted on the QVL machine publicly reachable at `*.qvl-project.com`, in HTTPS, **without opening a single port** on the home router and without exposing the home IP address.

> 💬 **Not a developer?** Normally, hosting a website at home means "opening a door" in your internet router — risky, and it reveals where you live (your IP). A tunnel works the other way around: **the machine itself opens a secure line *out* to Cloudflare**, like a permanent phone call. Visitors talk to Cloudflare, and Cloudflare relays their requests through that call. Your door stays closed; your address stays hidden.

## 🏗️ How it works

```mermaid
flowchart LR
    V["🌍 Visitor<br/>https://canopui.qvl-project.com"] -->|HTTPS| CF["☁️ Cloudflare<br/>DNS + TLS certificate<br/>+ DDoS protection"]
    CF <-->|"outbound-only tunnel<br/>(the machine calls out)"| CD["🖥️ QVL machine<br/><code>cloudflared</code> Windows service"]
    CD --> S0["🏠 ProjectCenter — apex<br/>localhost:8300"]
    CD --> S1["🎨 Vitrine CanopUI<br/>localhost:8081"]
    CD --> S2["📦 npm registry<br/>localhost:4873"]
    CD --> S3["🦊 GitLab CE<br/>localhost:8929"]
    CD --> S4["🏡 CustHome services<br/>localhost:3200-3203, 8083"]
    CD --> S5["🍸 Hobbies + PipeBoard<br/>localhost:8310-8314, 5191"]
    style CF fill:#fef0e6,stroke:#f38020,color:#000
    style CD fill:#e7f0ff,stroke:#1f6feb,color:#000
```

Key facts:

- **The domain** `qvl-project.com` is registered and managed at Cloudflare. One domain is enough for the whole ecosystem: every service gets its own **subdomain**.
- **The tunnel** is named `qvl-infra`. A tiny program, `cloudflared`, runs as a **Windows service** (starts with the machine) and keeps a permanent, encrypted, **outbound-only** connection to Cloudflare.
- **HTTPS is automatic.** Cloudflare terminates TLS at its edge with its own certificates — nothing to renew, ever.
- **Routing is declarative**: a single configuration file maps each public hostname to a local port (*ingress rules*).

## 🗺️ Exposed services

The real ingress list lives in `C:\Users\marti\.cloudflared\config.yml`. Current routes:

| Public URL | Local target | Service |
|---|---|---|
| `qvl-project.com` (apex) | `localhost:8300` | 🏠 QVL-ProjectCenter (React SPA, `serve-vitrine.mjs`) |
| `canopui.qvl-project.com` | `localhost:8081` | 🎨 CanopUI showcase (`serve-vitrine.mjs`, static build) |
| `npm.qvl-project.com` | `localhost:4873` | 📦 Private npm registry (Verdaccio) |
| `gitlab.qvl-project.com` | `localhost:8929` | 🦊 Self-hosted GitLab CE (active ingress) |
| `ch-auth.qvl-project.com` | `localhost:3200` | 🏡 CustHome — Authenticator (portal) |
| `ch-admin.qvl-project.com` | `localhost:3201` | 🏡 CustHome — Admin |
| `ch-drive.qvl-project.com` | `localhost:3202` | 🏡 CustHome — Drive |
| `ch-budgy.qvl-project.com` | `localhost:3203` | 🏡 CustHome — Budgy |
| `ch-relay.qvl-project.com` | `localhost:8083` | 🏡 CustHome — Relay MQTT-over-WebSocket |
| `tb-pipeboard.qvl-project.com` | `localhost:5191` | 🧰 ToolBox — PipeBoard |
| `api-cocktail.qvl-project.com` | `localhost:8310` | 🍸 Hobbies — HB-Api-Cocktail (Go) |
| `statbar.qvl-project.com` | `localhost:8312` | 🍸 Hobbies — HB-Front-StatBar |
| `fonddeshaker.qvl-project.com` | `localhost:8313` | 🍸 Hobbies — HB-Front-FondDeShaker |
| `departemental.qvl-project.com` | `localhost:8314` | 🍸 Hobbies — HB-Front-DeparteMental |

## ➕ Adding a new service

Three steps, ~2 minutes:

1. Make the service listen on a local port — e.g. `localhost:3001`.
2. Create the DNS route: `cloudflared tunnel route dns qvl-infra newapp.qvl-project.com`
3. Add an ingress block to the tunnel configuration file (`C:\Users\marti\.cloudflared\config.yml`), then restart the `cloudflared` service:

```yaml
- hostname: newapp.qvl-project.com
  service: http://localhost:3001
```

> ⚙️ **Restarting cloudflared** — `cloudflared` runs as a **Windows service**, so restarting it requires **UAC elevation**. From an elevated shell (or by launching one on the fly):
>
> ```powershell
> Start-Process powershell -Verb RunAs -ArgumentList '-Command','Restart-Service Cloudflared -Force'
> ```

> ⚠️ **Port discipline** — never reuse a port already taken by the infrastructure (see table above) or by GitLab's internal services. Each new app gets its own dedicated port.

> 🗂️ **Service scripts** — the runtime scripts that actually start these local services live in `C:\QVL\deploy\` (e.g. `serve-vitrine.mjs`, shared by the **CanopUI showcase (8081)** *and* the **ProjectCenter apex (8300)**; `serve-*.ps1`; the WSL `wsl-keepalive.ps1`). ⚠️ **This folder is not versioned** — a backup blind spot to address. It also holds a runtime secret (a Brevo API key) **in clear text**, to be secured.

## ❓ FAQ

<details>
<summary><b>Is the home IP address visible to visitors?</b></summary>
No. Visitors only ever see Cloudflare's servers. The machine's connection to Cloudflare is outbound, so the home IP is never published anywhere (it does not even need to be static).
</details>

<details>
<summary><b>What if the machine is off?</b></summary>
The public URLs answer with a Cloudflare error page until the machine (and the tunnel service) is back. Nothing to reconfigure — the tunnel reconnects automatically.
</details>

<details>
<summary><b>Is it free?</b></summary>
The tunnel and HTTPS are part of Cloudflare's free plan. The only cost is the domain name (~10 €/year).
</details>

</details>

<!-- ============================== FRANÇAIS ============================== -->
<details>
<summary><h3>🇫🇷&nbsp;&nbsp;Français</h3></summary>

<a id="-français"></a>

## 🎯 En une phrase

Un **Cloudflare Tunnel** rend les services hébergés sur la machine QVL accessibles publiquement sur `*.qvl-project.com`, en HTTPS, **sans ouvrir le moindre port** sur la box internet et sans exposer l'adresse IP du domicile.

> 💬 **Pas développeur ?** Normalement, héberger un site chez soi oblige à « ouvrir une porte » dans sa box internet — risqué, et cela révèle où l'on habite (son IP). Un tunnel fonctionne à l'envers : **c'est la machine elle-même qui ouvre une ligne sécurisée *vers* Cloudflare**, comme un appel téléphonique permanent. Les visiteurs parlent à Cloudflare, et Cloudflare relaie leurs demandes à travers cet appel. Ta porte reste fermée ; ton adresse reste cachée.

## 🏗️ Comment ça marche

```mermaid
flowchart LR
    V["🌍 Visiteur<br/>https://canopui.qvl-project.com"] -->|HTTPS| CF["☁️ Cloudflare<br/>DNS + certificat TLS<br/>+ protection DDoS"]
    CF <-->|"tunnel sortant uniquement<br/>(la machine appelle vers l'extérieur)"| CD["🖥️ Machine QVL<br/>service Windows <code>cloudflared</code>"]
    CD --> S0["🏠 ProjectCenter — apex<br/>localhost:8300"]
    CD --> S1["🎨 Vitrine CanopUI<br/>localhost:8081"]
    CD --> S2["📦 Registre npm<br/>localhost:4873"]
    CD --> S3["🦊 GitLab CE<br/>localhost:8929"]
    CD --> S4["🏡 Services CustHome<br/>localhost:3200-3203, 8083"]
    CD --> S5["🍸 Hobbies + PipeBoard<br/>localhost:8310-8314, 5191"]
    style CF fill:#fef0e6,stroke:#f38020,color:#000
    style CD fill:#e7f0ff,stroke:#1f6feb,color:#000
```

Les points clés :

- **Le domaine** `qvl-project.com` est enregistré et géré chez Cloudflare. Un seul domaine suffit pour tout l'écosystème : chaque service reçoit son propre **sous-domaine**.
- **Le tunnel** s'appelle `qvl-infra`. Un petit programme, `cloudflared`, tourne en **service Windows** (démarre avec la machine) et maintient une connexion permanente, chiffrée et **uniquement sortante** vers Cloudflare.
- **Le HTTPS est automatique.** Cloudflare termine le TLS sur ses serveurs avec ses propres certificats — rien à renouveler, jamais.
- **Le routage est déclaratif** : un unique fichier de configuration associe chaque nom d'hôte public à un port local (*règles d'ingress*).

## 🗺️ Services exposés

La liste d'ingress réelle vit dans `C:\Users\marti\.cloudflared\config.yml`. Routes actuelles :

| URL publique | Cible locale | Service |
|---|---|---|
| `qvl-project.com` (apex) | `localhost:8300` | 🏠 QVL-ProjectCenter (SPA React, `serve-vitrine.mjs`) |
| `canopui.qvl-project.com` | `localhost:8081` | 🎨 Vitrine CanopUI (`serve-vitrine.mjs`, build statique) |
| `npm.qvl-project.com` | `localhost:4873` | 📦 Registre npm privé (Verdaccio) |
| `gitlab.qvl-project.com` | `localhost:8929` | 🦊 GitLab CE auto-hébergé (ingress actif) |
| `ch-auth.qvl-project.com` | `localhost:3200` | 🏡 CustHome — Authenticator (portail) |
| `ch-admin.qvl-project.com` | `localhost:3201` | 🏡 CustHome — Admin |
| `ch-drive.qvl-project.com` | `localhost:3202` | 🏡 CustHome — Drive |
| `ch-budgy.qvl-project.com` | `localhost:3203` | 🏡 CustHome — Budgy |
| `ch-relay.qvl-project.com` | `localhost:8083` | 🏡 CustHome — Relay MQTT-over-WebSocket |
| `tb-pipeboard.qvl-project.com` | `localhost:5191` | 🧰 ToolBox — PipeBoard |
| `api-cocktail.qvl-project.com` | `localhost:8310` | 🍸 Hobbies — HB-Api-Cocktail (Go) |
| `statbar.qvl-project.com` | `localhost:8312` | 🍸 Hobbies — HB-Front-StatBar |
| `fonddeshaker.qvl-project.com` | `localhost:8313` | 🍸 Hobbies — HB-Front-FondDeShaker |
| `departemental.qvl-project.com` | `localhost:8314` | 🍸 Hobbies — HB-Front-DeparteMental |

## ➕ Ajouter un nouveau service

Trois étapes, ~2 minutes :

1. Faire écouter le service sur un port local — ex. `localhost:3001`.
2. Créer la route DNS : `cloudflared tunnel route dns qvl-infra newapp.qvl-project.com`
3. Ajouter un bloc d'ingress au fichier de configuration du tunnel (`C:\Users\marti\.cloudflared\config.yml`), puis redémarrer le service `cloudflared` :

```yaml
- hostname: newapp.qvl-project.com
  service: http://localhost:3001
```

> ⚙️ **Redémarrer cloudflared** — `cloudflared` tourne en **service Windows** : son redémarrage exige donc une **élévation UAC**. Depuis un shell élevé (ou en en lançant un à la volée) :
>
> ```powershell
> Start-Process powershell -Verb RunAs -ArgumentList '-Command','Restart-Service Cloudflared -Force'
> ```

> ⚠️ **Discipline des ports** — ne jamais réutiliser un port déjà pris par l'infrastructure (voir tableau ci-dessus) ou par les services internes de GitLab. Chaque nouvelle application reçoit son propre port dédié.

> 🗂️ **Scripts de service** — les scripts d'exécution qui démarrent réellement ces services locaux vivent dans `C:\QVL\deploy\` (ex. `serve-vitrine.mjs`, partagé par la **vitrine CanopUI (8081)** *et* l'**apex ProjectCenter (8300)** ; `serve-*.ps1` ; le `wsl-keepalive.ps1` côté WSL). ⚠️ **Ce dossier n'est pas versionné** — un angle mort de sauvegarde à traiter. Il contient aussi un secret runtime (une clé API Brevo) **en clair**, à sécuriser.

## ❓ FAQ

<details>
<summary><b>L'adresse IP du domicile est-elle visible par les visiteurs ?</b></summary>
Non. Les visiteurs ne voient jamais que les serveurs de Cloudflare. La connexion de la machine vers Cloudflare est sortante : l'IP du domicile n'est publiée nulle part (elle n'a même pas besoin d'être fixe).
</details>

<details>
<summary><b>Et si la machine est éteinte ?</b></summary>
Les URLs publiques répondent avec une page d'erreur Cloudflare jusqu'au retour de la machine (et du service tunnel). Rien à reconfigurer — le tunnel se reconnecte automatiquement.
</details>

<details>
<summary><b>Est-ce gratuit ?</b></summary>
Le tunnel et le HTTPS font partie du plan gratuit de Cloudflare. Le seul coût est le nom de domaine (~10 €/an).
</details>

</details>

---

<p align="center">
  <sub>© QVL — Documentation · <a href="../README.md">Hub</a> · Voir aussi : <a href="hebergement-gitlab.md">Hébergement GitLab</a> · <a href="../pipeline/ci-cd.md">CI/CD</a></sub><br>
  <sub>Crafted solo, with the help of agentic AI 🤖 · Conçu en solo, avec l'aide de l'IA agentique</sub>
</p>
