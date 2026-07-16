# Pipeline

Comment les pipelines CI/CD QVL fonctionnent : orchestrées par `gitlab.com`, calculées sur des runners auto-hébergés de la machine QVL (zéro minute gitlab.com consommée), du test jusqu'au déploiement et à la publication.

| Page | Sujet |
|---|---|
| [Pipeline CI/CD](ci-cd.md) | Flux complet d'un `git push` : `test` (lint + tests unitaires), `build` (librairie + vitrine), `deploy-vitrine` et `update-checkout` sur `main`, `publish` sur tag `vX.Y.Z`. Répartition orchestration (gitlab.com) / calcul (runners locaux), gestion des secrets via variables CI/CD masquées, et procédure de release. |

---

_Voir aussi : [Hébergement](../hebergement/README.md) · [Hub de documentation](../README.md)_
