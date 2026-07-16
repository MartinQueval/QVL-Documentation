# CanopUI

> Design system React de la flotte **QVL** — socle commun des portails, construit en wrapper au-dessus de **MUI**.

![package](https://img.shields.io/badge/package-canopui-cb3837?logo=npm&logoColor=white)
![version](https://img.shields.io/badge/version-1.1.1-blue)
![stack](https://img.shields.io/badge/React%2019%20·%20Vite%207%20·%20TypeScript-1f6feb)
![registry](https://img.shields.io/badge/registry-npm.qvl--project.com-4b5e40)

## Rôle

CanopUI est la **librairie de composants React partagée** entre les portails de l'écosystème QVL-Studio. Les portails n'importent **jamais MUI directement** : ils consomment uniquement `canopui`, qui expose des composants typés (TypeScript strict), accessibles et thématisables.

- **Wrapper MUI** — chaque composant est bâti sur MUI (thème, props), jamais réinventé.
- **Fork de [CH-UI-Library v0.7.0](../QVL-CustHome/README.md) (QVL-CustHome)** — librairie désormais indépendante, qui suit sa propre vie de version sans lien avec CustHome.
- **Design tokens** exposés à la fois en objet TypeScript typé et en **CSS variables `--ch-*`**, injectées dans le thème MUI (`cssVariables`) — surcharge possible sans rebuild.
- **Micro-frontends** — conçue pour être déclarée en `shared` singleton (Module Federation), chargée une seule fois pour tout l'écosystème.

## Stack

| Aspect | Choix |
|---|---|
| Framework | React 19 |
| Build | Vite 7 (library mode, ESM, types `.d.ts`) |
| Langage | TypeScript strict |
| Fondation | MUI 7 + Emotion (peer dependencies) |
| Tests | Vitest |
| Vitrine | Ladle |

## Installation

Le paquet **`canopui`** est publié sur le **registre npm privé QVL** (Verdaccio auto-hébergé) : `https://npm.qvl-project.com/` — lecture anonyme/publique. Voir le [guide du registre npm](../hebergement/registre-npm.md).

> Le registre n'est **pas** local. `localhost:4873` ne fonctionne que sur le serveur QVL lui-même. Depuis toute autre machine (dev, CI, agent IA) : utiliser exclusivement `https://npm.qvl-project.com/`.

```bash
# à la racine du projet consommateur
echo "registry=https://npm.qvl-project.com/" > .npmrc
npm install canopui
# peer dependencies si absentes du projet :
npm install react react-dom @mui/material @emotion/react @emotion/styled
```

Les consommateurs épinglent la version en **exact** dans leur `package.json`.

```tsx
import { ChThemeProvider, Button } from "canopui";
import "canopui/styles.css";

function App() {
  return (
    <ChThemeProvider>
      <Button variant="primary">Connexion</Button>
    </ChThemeProvider>
  );
}
```

## Composants & modules exportés

Tout est exposé depuis le point d'entrée unique `canopui` (voir `src/index.ts`).

### Actions

| Composant | Rôle |
|---|---|
| `Button` | Bouton principal (variantes `primary` / `secondary` / `danger`, tailles, état `loading`) |
| `AddButton`, `ApproveButton`, `EditButton`, `DeleteButton` | Boutons d'action sémantiques |
| `IconActionButton` | Bouton icône (variantes) |

### Saisie & formulaires

| Composant | Rôle |
|---|---|
| `Input`, `InputText`, `InputEmail`, `InputPassword` | Champs par type, icône intégrée, validation à la perte de focus |
| `PasswordStrength` | Indicateur de robustesse du mot de passe |
| `Checkbox` | Case à cocher (label `ReactNode`, `sublabel`, `error`, `required`) |
| `Toggle` | Interrupteur |
| `MultiSelect` | Sélection multiple |
| `Form` (+ `useForm`) | Formulaire complet (champs + erreur + soumission), logique extraite dans le hook |

### Structure & mise en page

| Composant | Rôle |
|---|---|
| `Layout`, `PageScaffold`, `PageContent` | Cadres de page portail |
| `Navbar` | Navigation (items, sous-items, footer mobile) |
| `SettingsMenu` | Menu de réglages (Popover) |
| `Stack` | Empilement avec espacement cohérent (tokens) |
| `Card`, `CardGrid` | Surfaces et grilles de cartes |
| `Divider`, `Separator` | Séparateurs |

### Contenu & typographie

| Composant | Rôle |
|---|---|
| `Heading` | Titres (niveau sémantique + taille visuelle découplés) |
| `Link` | Lien polymorphe (compatible react-router) |
| `BulletList`, `DescriptionList` | Listes à puces et paires terme/définition |
| `Icon` | Icônes SVG (`outline` / `solid`), recoloration via `currentColor` |

### Retour & feedback

| Composant | Rôle |
|---|---|
| `Feedback`, `Toast` | Messages `success` / `error` / `info` / `warning` |
| `Spinner` | Chargement (inline / pleine page) |
| `ProgressBar` | Barre de progression |
| `ConfirmDialog` | Boîte de confirmation |
| `SidePanel` | Panneau latéral |
| `StatusChip`, `Legend` | Puces de statut et légendes |

### Données & navigation

| Composant | Rôle |
|---|---|
| `DataTable` (+ `useInView`) | Table (en-têtes sticky, effet de scroll, cartes en mobile) |
| `Carousel` (+ `useCarousel`) | Carrousel |
| `Menu`, `MenuItem` | Menus positionnables |

### Upload de fichiers

| Élément | Rôle |
|---|---|
| `FileUploader`, `UploadDropZone` | Composants d'upload web |
| `useChunkedUpload`, `createUploader`, `createUploadTransport` | Cœur d'upload chunké (retry, planification des chunks) |

### Thème, i18n & décors

| Élément | Rôle |
|---|---|
| `ChThemeProvider`, `useChTheme`, `createChTheme`, `chTheme` | Thème clair/sombre au runtime |
| `ChI18nProvider`, `useTranslation` | Internationalisation (`fr` / `en`, changement au runtime) |
| `LanguageSelector` | Sélecteur de langue |
| `ShapeBackground` | Fond décoratif |

### Design tokens

`tokens`, `palette`, `paletteDark`, `typography`, `spacing`, `radius`, `shadows` — palette, typographie, espacements, radius et ombres, en objet TypeScript typé et en CSS variables `--ch-*`.

### Auth, HTTP & validation

| Élément | Rôle |
|---|---|
| `CurrentUserProvider`, `useCurrentUser`, `RouteGuard`, `createRouteGuard`, `useRouteGuard` | Contexte utilisateur et gardes de routes |
| `buildLoginUrl`, `buildCguUrl`, `navigateTo` | Helpers de navigation/auth |
| `createApiClient`, `ApiError` | Client HTTP |
| `isValidEmail`, `isValidName`, `isValidPassword`, `passwordStrength`, `EMAIL_REGEX`… | Validation partagée |

## Theming

Le thème est fourni par `ChThemeProvider` (ThemeProvider MUI + CssBaseline). Les composants résolvent leurs couleurs via `var(--ch-…)` au runtime : aucune valeur n'est figée dans le rendu, ce qui rend le thème **surchargeable sans rebuild**.

```tsx
<ChThemeProvider defaultMode="system">
  <App />
</ChThemeProvider>
```

Le hook `useChTheme()` renvoie `{ mode, resolvedMode, setMode, toggleMode }` : `mode` est la préférence (`light` / `dark` / `system`), `resolvedMode` le mode effectif (`system` résolu via `prefers-color-scheme`). La préférence est persistée dans `localStorage` (clé `ch-theme-mode`).

Surcharger 2-3 variables `--ch-*` dans une feuille chargée après la lib suffit à changer le rendu (dark mode / white-label), sans recompiler. Détails : voir `docs/THEMING.md` du dépôt.

## Vitrine

Les composants sont présentés dans la vitrine **Ladle**, déployée sur **[canopui.qvl-project.com](https://canopui.qvl-project.com)** (exposée via le [tunnel Cloudflare](../hebergement/tunnel-cloudflare.md)). En local : `npm run ladle`.

## Versioning & release

CanopUI suit le **semver** :

- **MAJOR** — rupture d'API publique (props renommées/supprimées, comportement changé)
- **MINOR** — nouveau composant, nouvelle prop, nouveau token
- **PATCH** — correction de bug sans changement d'API

La **publication est automatique** : pousser un tag `vX.Y.Z` sur GitLab déclenche la [pipeline CI](../pipeline/README.md), qui builde puis publie `canopui@X.Y.Z` sur Verdaccio (authentification par le secret `VERDACCIO_TOKEN`, jamais de publication manuelle depuis un poste). Une version publiée ne se dépublie pas : en cas de régression, publier un nouveau patch. Procédure détaillée : `docs/RELEASE.md` du dépôt.

## Sécurité CI

La CI CanopUI tourne sur un runner GitLab **shell executor** (WSL). Les jobs sensibles sont restreints aux refs de confiance (`publish` sur tag `v*` protégé ; `deploy-vitrine` / `update-checkout` sur `main` protégée), les includes sont épinglés sur SHA, et le token de registre est une variable CI **Masked + Protected** écrite hors du checkout. Scans de secrets (`gitleaks`) et de vulnérabilités (`osv-scanner`, advisory) sur tout l'historique. Détails : `SECURITY.md` du dépôt.

## Développement

```bash
npm install
npm run ladle    # vitrine des composants
npm test         # tests unitaires
npm run build    # build de la lib (dist/)
```

## Liens utiles

- Vitrine : [canopui.qvl-project.com](https://canopui.qvl-project.com)
- Registre npm : [guide du registre](../hebergement/registre-npm.md)
- Pipeline CI/CD : [documentation pipeline](../pipeline/README.md)
- Dépôt miroir GitHub : <https://github.com/QVL-Studio/CanopUI>

---

_Documentation maintenue par la flotte QVL-Studio · [Hub de documentation](../README.md)_
