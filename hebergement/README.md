# Hébergement

Comment l'infrastructure QVL est hébergée, exposée et sauvegardée : GitLab (source de vérité + miroir), registre npm privé et tunnel Cloudflare. Tout tourne sur la machine QVL, sans port ouvert sur la box.

| Page | Sujet |
|---|---|
| [Hébergement GitLab](hebergement-gitlab.md) | Double instance GitLab : `gitlab.com` (source de vérité) et un GitLab CE auto-hébergé (miroir de sauvegarde, CI désactivée), synchronisés par push mirror automatique. Tourne sous WSL2 / Ubuntu. |
| [Registre npm privé](registre-npm.md) | Registre Verdaccio sur `npm.qvl-project.com` : lecture anonyme (installation sans compte), publication réservée à la CI. Piège classique `localhost:4873` vs URL publique, relais de npmjs.org. |
| [Tunnel Cloudflare](tunnel-cloudflare.md) | Tunnel `qvl-infra` (`cloudflared`) exposant les services `*.qvl-project.com` en HTTPS, sans ouvrir de port ni révéler l'IP du domicile. Table des services exposés et procédure d'ajout. |

---

_Voir aussi : [Pipeline CI/CD](../pipeline/README.md) · [Hub de documentation](../README.md)_
