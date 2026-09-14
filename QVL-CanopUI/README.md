# CanopUI

> Design system React de la flotte **QVL** — socle commun des portails, construit en wrapper au-dessus de **MUI**.

![package](https://img.shields.io/badge/package-canopui-cb3837?logo=npm&logoColor=white)
![version](https://img.shields.io/badge/version-3.0.1-blue)
![stack](https://img.shields.io/badge/React%2019%20·%20Vite%207%20·%20TypeScript-1f6feb)
![registry](https://img.shields.io/badge/registry-npm.qvl--project.com-4b5e40)

> 🎨 **Direction Artistique** — le langage visuel complet (principes, palette clair/sombre, typographie, espacements, rayons, élévation, mouvement, responsive, décors) est documenté dans **[direction-artistique.md](direction-artistique.md)**.

## Rôle

CanopUI est la **librairie de composants React partagée** entre les portails de l'écosystème QVL-Studio. Les portails n'importent **jamais MUI directement** : ils consomment uniquement `canopui`, qui expose des composants typés (TypeScript strict), accessibles et thématisables.

- **Wrapper MUI** — chaque composant est bâti sur MUI (thème, props), jamais réinventé.
- **Fork de [CH-UI-Library v0.7.0](../QVL-CustHome/README.md) (QVL-CustHome)** — librairie désormais indépendante, qui suit sa propre vie de version sans lien avec CustHome.
- **Design tokens** exposés à la fois en objet TypeScript typé et en **CSS variables `--canop-*`** (préfixe `cssVarPrefix: "canop"`), injectées dans le thème MUI (`cssVariables`) — surcharge possible sans rebuild.
- **Micro-frontends** — conçue pour être déclarée en `shared` singleton (Module Federation), chargée une seule fois pour tout l'écosystème.

## Stack

| Aspect | Choix |
|---|---|
| Framework | React 19 |
| Build | Vite 7 (library mode, ESM, types `.d.ts`) |
| Langage | TypeScript strict |
| Fondation | MUI 9 + Emotion (peer dependencies) |
| Animation | framer-motion (peer dependency, non optionnelle depuis la 3.0) |
| Cartographie | MapLibre GL (peer dependency **optionnelle**, sous-chemin `canopui/map`) |
| Tests | Vitest |
| Vitrine | Site maison (construit avec les composants CanopUI) |

## Installation

Le paquet **`canopui`** est publié sur le **registre npm privé QVL** (Verdaccio auto-hébergé) : `https://npm.qvl-project.com/` — lecture anonyme/publique. Voir le [guide du registre npm](../hebergement/registre-npm.md).

> Le registre n'est **pas** local. `localhost:4873` ne fonctionne que sur le serveur QVL lui-même. Depuis toute autre machine (dev, CI, agent IA) : utiliser exclusivement `https://npm.qvl-project.com/`.

```bash
# à la racine du projet consommateur
echo "registry=https://npm.qvl-project.com/" > .npmrc
npm install canopui
# peer dependencies si absentes du projet :
npm install react react-dom @mui/material @emotion/react @emotion/styled framer-motion
# peer dependency OPTIONNELLE — uniquement si le portail affiche une carte (canopui/map) :
npm install maplibre-gl
```

> Depuis la **3.0**, `framer-motion` (`^13`) est une **peer dependency non optionnelle** : installer `canopui` ne l'installe plus, il faut l'ajouter au projet consommateur, sinon la construction casse (le cœur — `Navbar`, `SidePanel`, `Carousel`… — l'importe). `maplibre-gl` (`^6`) reste **optionnelle** : requise seulement pour le sous-chemin `canopui/map`.

Les consommateurs épinglent la version en **exact** dans leur `package.json`.

```tsx
import { CanopThemeProvider, Button } from "canopui";
import "canopui/styles.css";

function App() {
  return (
    <CanopThemeProvider>
      <Button variant="primary">Connexion</Button>
    </CanopThemeProvider>
  );
}
```

> Un portail qui affiche une **carte** ou qui utilise le **fond de canopée** (décor par défaut de `PageScaffold`, voir plus bas) doit poser le greffon `canopyVideo()` dans sa configuration Vite — il sert les scènes vidéo **et** les fichiers du worker de MapLibre :
>
> ```ts
> import { canopyVideo } from "canopui/vite";
>
> export default defineConfig({ plugins: [react(), canopyVideo()] });
> ```

## Composants & modules exportés

L'essentiel est exposé depuis le point d'entrée unique `canopui` (voir `src/index.ts`). Deux **sous-chemins** isolent le code lourd ou à peer dependency optionnelle :

| Sous-chemin | Contenu |
|---|---|
| `canopui` | Tous les composants, hooks, tokens, thème et helpers (point d'entrée principal) |
| `canopui/map` | Composant `Map` (MapLibre GL), `buildCanopMapStyle`, `resolveMapTiles`, `CANOP_MAP_TILES`… — peer dependency `maplibre-gl` optionnelle |
| `canopui/vite` | Greffon `canopyVideo()` pour la configuration Vite du consommateur (voir Installation) |
| `canopui/styles.css` | Feuille de styles à importer une fois |

### Actions

| Composant | Rôle |
|---|---|
| `Button` | Bouton principal (variantes `primary` / `secondary` / `danger` / `text` / `ghost`, tailles, état `loading`) |
| `AddButton`, `ApproveButton`, `EditButton`, `DeleteButton` | Boutons d'action sémantiques (`size` en `string`, `iconSize` explicite depuis la 3.0) |
| `IconActionButton` | Bouton icône (variantes) |
| `SaveButton` (+ `useSaveButton`) | Bouton d'enregistrement avec états (`idle`/`saving`/`saved`/`error`) |
| `Pressable` | Primitive cliquable générique (rôles, rayons, espacement tokenisés) |

### Saisie & formulaires

| Composant | Rôle |
|---|---|
| `Input`, `InputText`, `InputEmail`, `InputPassword` | Champs par type, icône intégrée, validation à la perte de focus |
| `PasswordStrength` | Indicateur de robustesse du mot de passe |
| `InlineInput` | Champ éditable en ligne (sur `background.paper`) |
| `Checkbox` | Case à cocher (label `ReactNode`, `sublabel`, `error`, `required`) |
| `Choice` | Choix unique (états stylés) |
| `Toggle` | Interrupteur |
| `Select` | Liste déroulante simple |
| `MultiSelect` | Sélection multiple |
| `Autocomplete` (+ `foldForSearch`) | Champ à autocomplétion (recherche insensible aux accents) |
| `Rating` | Notation par étoiles |
| `Form` (+ `useForm`) | Formulaire complet (champs + erreur + soumission), logique extraite dans le hook |

### Structure & mise en page

| Composant | Rôle |
|---|---|
| `Layout`, `PageScaffold`, `PageContent` | Cadres de page portail (fond `canopy` par défaut sur `PageScaffold`) |
| `Navbar` | Navigation (items, sous-items, footer mobile) |
| `Breadcrumb` (+ `useBreadcrumb`) | Fil d'Ariane |
| `Toolbar` (+ `useToolbar`) | Barre d'outils (recherche, tri, bascule de vue) |
| `SelectionBar` (+ `useSelectionBar`) | Barre d'actions de sélection |
| `CommandPalette` (+ `useCommandPalette`) | Palette de commandes (⌘K) |
| `SettingsMenu` | Menu de réglages (Popover) |
| `CookieBanner` (+ `useCookieConsent`) | Bandeau de consentement cookies |
| `LegalLinks` | Liens légaux (mentions, CGU…) |
| `Stack` | Empilement avec espacement cohérent (tokens) ; props `sticky` / `stickyOffset` |
| `Card`, `CardGrid` | Surfaces (translucides + `backdrop-filter` depuis la 3.0) et grilles (`columns` responsive) |
| `Divider`, `Separator` | Séparateurs |

### Contenu & typographie

| Composant | Rôle |
|---|---|
| `Heading` | Titres (niveau sémantique + taille visuelle découplés ; `size` responsive par point de rupture) |
| `Text` | Texte tokenisé (variantes `body-*`/`label`/`caption`…, `tone`, `weight`, `tabularNums`, `slashedZero`) |
| `Link` | Lien polymorphe (compatible react-router) |
| `Badge` | Pastille de comptage / statut |
| `BulletList`, `DescriptionList` | Listes à puces et paires terme/définition |
| `Icon` | Icônes SVG (`outline` / `solid`), recoloration via `currentColor` |

### Retour & feedback

| Composant | Rôle |
|---|---|
| `Feedback`, `Toast` | Messages `success` / `error` / `info` / `warning` |
| `Spinner` | Chargement (inline / pleine page) |
| `ProgressBar` | Barre de progression (simple **ou** segmentée — types disjoints depuis la 3.0) |
| `ConfirmDialog` | Boîte de confirmation |
| `SidePanel` | Panneau latéral |
| `EmptyState` | État vide (illustration + message + action) |
| `StatusChip`, `Legend` | Puces de statut et légendes (entrées `tone` **ou** `color` libre) |
| `Countdown` | Compte à rebours |
| `Lives`, `Streak`, `ShareResult` | Composants ludiques (vies, série, partage de résultat) |

### Données & navigation

| Composant | Rôle |
|---|---|
| `DataTable` (+ `useInView`) | Table (en-têtes sticky, effet de scroll, cartes en mobile) |
| `Carousel` (+ `useCarousel`) | Carrousel indexé (une carte à la fois, `arrows`/`bars`, refonte 3.0) |
| `SegmentedControl` (+ `SlidingIndicator`) | Onglets exclusifs à indicateur glissant |
| `StatCard` | Carte de chiffre clé |
| `Donut` (+ `useDonut`) | Anneau de progression |
| `Menu`, `MenuItem` (+ `useMenuAnchor`) | Menus positionnables |
| `Map` *(sous-chemin `canopui/map`)* | Carte vectorielle MapLibre GL (tuiles OpenFreeMap) |
| `SvgMap` (+ `SvgMapControls`, `useSvgMapGestures`) | Carte SVG interactive (ex. départements) |

### Upload de fichiers

| Élément | Rôle |
|---|---|
| `FileUploader`, `UploadDropZone`, `Dropzone` | Composants d'upload web |
| `FileCard` | Carte de fichier (aperçu, sélection) |
| `Lightbox` | Visionneuse d'images |
| `useChunkedUpload`, `createUploader`, `createUploadTransport` | Cœur d'upload chunké (retry, planification des chunks) |

### Thème, i18n & décors

| Élément | Rôle |
|---|---|
| `CanopThemeProvider`, `useCanopTheme`, `createCanopTheme`, `canopTheme` | Thème clair/sombre au runtime |
| `ThemeToggle` (+ `useThemeToggle`) | Bascule Clair / Sombre |
| `CanopI18nProvider`, `useTranslation` | Internationalisation (`fr` / `en`, changement au runtime ; pluriels via `Intl.PluralRules` depuis la 3.0) |
| `LanguageSelector` | Sélecteur de langue |
| `SoundToggle` (+ `useSoundToggle`) | Bascule du son |
| `CanopyBackground`, `canopyScenesFrom`, `resolveCanopyScene` | Fond vidéo « canopée » (décor par défaut de `PageScaffold`) |
| `ShapeBackground` | Fond décoratif à formes organiques (repli `background="shapes"`) |

### Design tokens

`tokens`, `palette`, `paletteDark`, `typography`, `spacing`, `radius`, `shadows` — palette, typographie, espacements, radius et ombres, en objet TypeScript typé et en CSS variables `--canop-*`. La constante de version est exportée sous `CANOP_VERSION`.

### Auth, HTTP & validation

| Élément | Rôle |
|---|---|
| `CurrentUserProvider`, `useCurrentUser`, `RouteGuard`, `createRouteGuard`, `useRouteGuard` | Contexte utilisateur et gardes de routes |
| `buildLoginUrl`, `buildCguUrl`, `navigateTo` | Helpers de navigation/auth |
| `createApiClient`, `ApiError` | Client HTTP |
| `isValidEmail`, `isValidName`, `isValidPassword`, `passwordStrength`, `EMAIL_REGEX`… | Validation partagée |

## Theming

Le thème est fourni par `CanopThemeProvider` (ThemeProvider MUI + CssBaseline). Les composants résolvent leurs couleurs via `var(--canop-…)` au runtime : aucune valeur n'est figée dans le rendu, ce qui rend le thème **surchargeable sans rebuild**.

```tsx
<CanopThemeProvider defaultMode="system">
  <App />
</CanopThemeProvider>
```

Le hook `useCanopTheme()` renvoie `{ mode, resolvedMode, setMode, toggleMode }` : `mode` est la préférence (`light` / `dark` / `system`), `resolvedMode` le mode effectif (`system` résolu via `prefers-color-scheme`). La préférence est persistée dans `localStorage` (clé `canop-theme-mode`).

Surcharger 2-3 variables `--canop-*` dans une feuille chargée après la lib suffit à changer le rendu (dark mode / white-label), sans recompiler. Détails : voir `docs/THEMING.md` du dépôt.

> **Rupture 3.0** — le rebrand `Ch*` → `Canop*` est **complet et sans alias de rétrocompatibilité** : symboles (`ChThemeProvider` → `CanopThemeProvider`…), CSS variables (`--ch-*` → `--canop-*`), clés i18n (`ch.*` → `canop.*`), clés `localStorage` (`ch-theme-mode` → `canop-theme-mode`, sans migration) et constante de version (`CH_UI_VERSION` → `CANOP_VERSION`). Voir le `CHANGELOG.md` du dépôt pour le codemod et les ruptures silencieuses.

## Nouveautés 3.x

La **3.0** est une version majeure (dette technique + typographie + squircles) ; la **3.0.1** corrige deux défauts visibles seulement en production (worker MapLibre non émis, champs pré-remplis). Points saillants côté consommateur :

- **Cartographie vectorielle** — le sous-module `canopui/map` expose un composant `Map` bâti sur **MapLibre GL** (fini Leaflet), avec `buildCanopMapStyle`, `resolveMapTiles` et `CANOP_MAP_TILES` (tuiles **OpenFreeMap**). `Map` gère la perte de contexte WebGL (repli lisible) ; `SvgMap` reste disponible pour les cartes SVG.
- **Fond de canopée** — `CanopyBackground` (+ `canopyScenesFrom`, `resolveCanopyScene`) est le **décor par défaut de `PageScaffold`** depuis la 3.0 (prop `background="canopy"`). Deux scènes vidéo (claire / sombre), ~32 Mo au total, servies comme fichiers. Repli sur l'ancien décor via `background="shapes"` (`ShapeBackground` reste exporté).
- **Greffon Vite `canopyVideo()`** — sous-module `canopui/vite`. Il sert les scènes de canopée **et**, depuis la 3.0.1, les fichiers du **worker de MapLibre** (que l'empaqueteur n'émet pas), à la même adresse en développement et au build. Requis dès qu'un portail affiche une carte ou le fond de canopée.
- **Typographie** — police d'affichage **Titan One** pour les titres (repli `Chivo`), crénage optique, graisses étendues, variantes numériques et nouveau composant `Text`.
- **Squircles** — tous les coins arrondis deviennent des **superellipses** ; bordures et ombres reconstruites (`squircleSurface`).
- **Peer deps** — `framer-motion` (`^13`) devient peer **non optionnelle** ; `maplibre-gl` (`^6`) peer **optionnelle** ; `@mui/material` passe en **`^9`**.

## Vitrine

Les composants sont présentés dans la **vitrine maison** (`vitrine/`, construite avec les composants CanopUI eux-mêmes), déployée sur **[canopui.qvl-project.com](https://canopui.qvl-project.com)** (exposée via le [tunnel Cloudflare](../hebergement/tunnel-cloudflare.md)). En local : `npm run vitrine` (port 61000).

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
npm run vitrine  # vitrine des composants (port 61000)
npm test         # tests unitaires
npm run build    # build de la lib (dist/)
```

## Liens utiles

- **Direction Artistique** : [direction-artistique.md](direction-artistique.md) — principes, langage visuel et tokens de référence
- Vitrine : [canopui.qvl-project.com](https://canopui.qvl-project.com)
- Registre npm : [guide du registre](../hebergement/registre-npm.md)
- Pipeline CI/CD : [documentation pipeline](../pipeline/README.md)
- Dépôt miroir GitHub : <https://github.com/QVL-Studio/CanopUI>

---

_Documentation maintenue par la flotte QVL-Studio · [Hub de documentation](../README.md)_
