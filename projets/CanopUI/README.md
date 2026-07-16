# CanopUI

<img src="logo.png" alt="Logo CanopUI" width="64" height="64" />

![package](https://img.shields.io/badge/package-canopui-cb3837) ![version](https://img.shields.io/badge/version-1.1.1-blue) ![registry](https://img.shields.io/badge/registry-npm.qvl--project.com-4b5e40)

> Design system React de la flotte **QVL**, socle commun de tous les portails.

CanopUI est la librairie de composants React partagée de l'écosystème QVL-Studio. C'est un **wrapper au-dessus de MUI** (les portails n'importent jamais MUI directement), typé en TypeScript strict, avec thème clair/sombre au runtime et design tokens exposés en CSS variables `--ch-*`. Fork indépendant de **CH-UI-Library v0.7.0** (QVL-CustHome). Distribuée via le registre npm privé **Verdaccio** (`npm.qvl-project.com`) et présentée dans une vitrine **Ladle** sur [canopui.qvl-project.com](https://canopui.qvl-project.com).

## Sommaire

- [Installation](#installation)
- [En bref](#en-bref)
- [Documentation complète](#documentation-complète)

## Installation

```bash
echo "registry=https://npm.qvl-project.com/" > .npmrc
npm install canopui
```

Les consommateurs épinglent la version en **exact**. La publication est automatique via la CI sur un tag `vX.Y.Z`.

## En bref

- **Composants** : boutons, champs de saisie & formulaires, structure de page (`Layout`, `PageScaffold`, `Navbar`…), `Card`, `DataTable`, `Carousel`, feedback (`Toast`, `Feedback`, `Spinner`), upload de fichiers, et plus.
- **Thème** : `ChThemeProvider` + hook `useChTheme()` (clair/sombre/système, persistance `localStorage`).
- **i18n** : `ChI18nProvider` + `useTranslation` (fr/en au runtime).

## Documentation complète

La documentation de référence — rôle, installation, liste exhaustive des composants exportés, theming, versioning/release et sécurité CI — est ici :

**→ [Documentation CanopUI](../../QVL-CanopUI/README.md)**

## Liens utiles

- [Guide d'hébergement](../../hebergement/README.md)
- [Pipeline CI/CD](../../pipeline/README.md)
- Dépôt miroir : <https://github.com/QVL-Studio/CanopUI>

---

_Documentation maintenue par la flotte QVL-Studio._
