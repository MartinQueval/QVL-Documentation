# Direction Artistique — CanopUI

> La **DA de référence** du design system **CanopUI** : principes, langage visuel et tokens qui régissent tous les portails de la flotte QVL. Toute nouvelle interface s'y conforme ; tout écart doit être justifié.

- **Version couverte** : `canopui@2.0.1`
- **Fondation** : wrapper **MUI 7** — les portails n'importent jamais MUI directement, uniquement `canopui`.
- **Source de vérité des valeurs** : `src/tokens/` du dépôt CanopUI (objet TypeScript typé + CSS variables `--ch-*`).
- **Vitrine vivante** : [canopui.qvl-project.com](https://canopui.qvl-project.com) — démos live, code copiable, tables de props.

---

## 1. Le parti pris — « Terrain vivant »

CanopUI transforme les portails en un **terrain vivant** : des surfaces en profondeur, un mouvement organique et des fondations entièrement tokenisées. Trois intentions guident chaque décision :

1. **Naturel et organique** — palette végétale (verts profonds, or, sauge), courbes douces, animations qui décélèrent comme un objet réel. Rien de mécanique ni de brutal.
2. **Profondeur lisible** — la hiérarchie passe par l'élévation (ombres), les surfaces (`sunken` → `overlay`) et l'espace, jamais par des bordures dures ou des couleurs criardes.
3. **Cohérence par les tokens** — aucune couleur, taille ou durée n'est écrite en dur dans un composant. Tout traverse les tokens et les variables `--ch-*`, ce qui rend le thème **surchargeable sans rebuild**.

---

## 2. Principes fondateurs (non négociables)

| Principe | Règle |
|---|---|
| **Action = VERT** | La couleur d'action primaire est le **vert** (`primary`). Le bouton principal, les liens actifs, les indicateurs de sélection sont verts. |
| **L'OR est un accent rare** | L'or (`accent`, `#f4ad15`) souligne l'exceptionnel (favicons, mise en avant ponctuelle). Jamais comme couleur d'action courante, jamais en aplat massif. |
| **Light + Dark obligatoires** | Tout composant rend correctement en clair **et** en sombre. Aucune couleur/ombre en dur : uniquement des rôles de palette résolus au runtime. |
| **Mouvement organique** | Les transitions utilisent des courbes naturelles (`organic`, `spring-soft`, `decelerate`). Pas d'easing linéaire ni de saut. |
| **Reduced-motion respecté** | `prefers-reduced-motion: reduce` neutralise animations et transitions (durées ~0) ; les décors animés deviennent statiques. |
| **Responsive mobile-first** | 3 paliers (mobile / tablette / desktop). Jamais de scroll horizontal ; cibles tactiles confortables ; `safe-area` gérée. |
| **Zéro valeur en dur** | Couleurs → rôles de palette ; tailles → `spacing`/`radius` ; durées/courbes → `motion`. Les littéraux `px`/hex/`ms` en composant sont proscrits. |
| **Wrapper, pas fork** | Chaque composant est bâti sur MUI (thème, props), jamais réinventé. MUI n'est jamais exposé aux portails. |

---

## 3. Couleur

Deux palettes complètes (clair / sombre), organisées par **rôles**. Un composant nomme toujours un rôle (`primary.main`, `text.secondary`, `surface.raised`…), jamais une valeur hexadécimale.

### 3.1 Palette claire (`palette`)

| Rôle | main | light | dark | contrastText |
|---|---|---|---|---|
| **primary** (action, vert) | `#1e6244` | `#2c8a61` | `#154731` | `#fbfaf9` |
| **secondary** (sauge) | `#8f9a74` | `#bcc2a8` | `#6a7455` | `#0b0f08` |
| **accent** (or, rare) | `#f4ad15` | `#f7c24f` | `#c98c0a` | `#241a03` |
| **success** | `#2e7d32` | `#e7f4e8` | `#1b5e20` | `#ffffff` |
| **warning** | `#b26a00` | `#fdf0dd` | `#8a5200` | `#ffffff` |
| **error** | `#b3261e` | `#fbe9e7` | `#8c1d17` | `#ffffff` |
| **info** | `#1a4f8b` | `#e6f0fb` | `#123a66` | `#ffffff` |

**Fonds & surfaces (clair)**

| Clé | Valeur | Usage |
|---|---|---|
| `background.default` | `#f7f4ef` | Fond de page (crème chaud) |
| `background.paper` | `#ffffff` | Cartes, feuilles |
| `surface.sunken` | `#efece5` | Creux (zones en retrait) |
| `surface.base` | `#fbfaf9` | Surface neutre |
| `surface.raised` | `#ffffff` | Surface élevée |
| `surface.overlay` | `#ffffff` | Overlays, menus |
| `divider` | `#e6e1d8` | Séparateurs |

**Texte (clair)** : `primary #14100b` · `secondary #5c574d` · `disabled #9b968c` · `onPrimary #fbfaf9` · `onAccent #241a03`

### 3.2 Palette sombre (`paletteDark`)

En sombre, le **vert s'inverse** : la couleur d'action devient une **menthe claire** lisible sur fond sombre. L'**or reste identique** (accent stable dans les deux modes).

| Rôle | main | light | dark | contrastText |
|---|---|---|---|---|
| **primary** (menthe) | `#9de1c4` | `#c2eeda` | `#6fbf9e` | `#08130d` |
| **secondary** | `#a9b48c` | `#c6cfad` | `#7f8a63` | `#0b0f08` |
| **accent** (or, inchangé) | `#f4ad15` | `#f7c24f` | `#c98c0a` | `#241a03` |
| **success** | `#81c784` | `#122b14` | `#2e7d32` | `#08130d` |
| **warning** | `#e0a542` | `#2e2109` | `#b26a00` | `#1a1204` |
| **error** | `#e2726b` | `#2e1210` | `#b3261e` | `#1a0908` |
| **info** | `#7aaede` | `#0e2137` | `#1a4f8b` | `#08111c` |

> Note : en sombre, `light` et `dark` d'un rôle sont souvent **inversés** (le `light` sert de teinte de fond sourde, ex. `success.light #122b14`). C'est voulu pour les fonds teintés discrets (chips, badges).

**Fonds & surfaces (sombre)**

| Clé | Valeur |
|---|---|
| `background.default` | `#0c0f0d` |
| `background.paper` | `#1b1f1c` |
| `surface.sunken` | `#0f1210` |
| `surface.base` | `#151815` |
| `surface.raised` | `#1b1f1c` |
| `surface.overlay` | `#232823` |
| `divider` | `#2c322d` |

**Texte (sombre)** : `primary #f3f1ec` · `secondary #b8b3a7` · `disabled #797469` · `onPrimary #08130d` · `onAccent #241a03`

### 3.3 Règles d'usage de la couleur

- **Vert = action.** Bouton primaire, lien actif, indicateur d'onglet, focus.
- **Or = accent rare.** Jamais pour une action courante. Exception documentée : le bouton de déconnexion est **or en clair** / **noir (texte vert) en sombre** (le vert clair du dark ne « matchait » pas l'or — décision Martin).
- **Teintes sémantiques** : `success`/`warning`/`error`/`info` réservées au feedback, pas à la décoration.
- **Fonds teintés** : utiliser les `.light` des rôles (clair) ou leurs équivalents sombres, jamais un `rgba()` en dur.

---

## 4. Typographie

**Police** : `Chivo` (grotesque géométrique), avec repli `system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif`.

**Poids** : `regular 400` · `medium 500` · `semibold 600` · `bold 700`
**Interlignes** : `tight 1.15` · `snug 1.3` · `normal 1.5` · `relaxed 1.7`

### 4.1 Échelle « display » (héros, marketing)

| Token | Taille | Poids | Interligne |
|---|---|---|---|
| `display-1` | `3.815rem` | bold | tight |
| `display-2` | `2.863rem` | bold | tight |
| `display-3` | `2.145rem` | bold | tight |
| `display-4` | `1.61rem` | semibold | snug |

### 4.2 Échelle applicative (`app`)

| Token | Taille | Poids | Usage |
|---|---|---|---|
| `title-lg` | `1.5rem` | bold | Titre de page |
| `title-md` | `1.25rem` | semibold | Titre de section |
| `title-sm` | `1.0625rem` | semibold | Sous-titre |
| `body-lg` | `1rem` | regular | Corps principal |
| `body-md` | `0.9375rem` | regular | Corps dense |
| `body-sm` | `0.8125rem` | regular | Corps secondaire |
| `label` | `0.8125rem` | semibold | Libellés de champ |
| `caption` | `0.75rem` | regular | Légendes |
| `overline` | `0.6875rem` | semibold | Sur-titres (majuscules) |
| `metric` | `1.75rem` | bold | Chiffres clés (StatCard) |

### 4.3 Titres MUI (`h1`–`h5`)

Échelle modulaire (ratio ~1.333) : `h1 4.21rem` · `h2 3.158rem` · `h3 2.369rem` · `h4 1.777rem` · `h5 1.333rem`. Tous en `bold`, interligne `tight`.

> Le composant `Heading` **découple** le niveau sémantique (`level`, pour l'accessibilité) de la taille visuelle (`size`).

---

## 5. Espacement, rayons, élévation

### 5.1 Espacement (`spacing`) — base 8

| Token | Valeur | | Token | Valeur |
|---|---|---|---|---|
| `3xs` | `0.125rem` | | `lg` | `1.5rem` |
| `2xs` | `0.25rem` | | `xl` | `2rem` |
| `xs` | `0.5rem` | | `2xl` | `3rem` |
| `sm` | `0.75rem` | | `3xl` | `4rem` |
| `md` | `1rem` | | `4xl` | `6rem` |

`spacing.unit = 8` : le `spacing()` de MUI est calé sur 8 px.

### 5.2 Rayons (`radius`)

| Token | Valeur | Usage |
|---|---|---|
| `xs` | `0.25rem` | Éléments fins |
| `sm` | `0.5rem` | Badges, chips |
| `md` | `0.75rem` | **Défaut** (borderRadius du thème) |
| `lg` | `1rem` | Cartes |
| `xl` | `1.375rem` | Cartes flottantes, panneaux |
| `2xl` | `1.75rem` | Bottom-sheets (coin haut) |
| `pill` | `62.4375rem` | Formes en pilule (toggles, chips ronds) |

### 5.3 Élévation (`shadows`, `e1`→`e5`)

L'élévation porte la profondeur. En **sombre**, chaque ombre ajoute un **liseré lumineux vert** (`rgba(157,225,196, …)`) pour détacher les surfaces sur fond noir, et `e5` un léger halo.

| Niveau | Clair | Sombre (résumé) |
|---|---|---|
| `e1` | `0 1px 2px …0.06, 0 1px 3px …0.10` | ombre + liseré vert 4 % |
| `e2` | `0 2px 6px …0.08, 0 4px 12px …0.10` | + liseré vert 5 % |
| `e3` | `0 6px 16px …0.12` | + liseré vert 6 % |
| `e4` | `0 12px 28px …0.16` | + liseré vert 7 % |
| `e5` | `0 20px 48px …0.22` | + liseré vert 8 % + halo |

Mappées sur `theme.shadows[1..5]` — un composant écrit `elevation={3}` ou `var(--ch-shadow-e3)`, jamais une ombre en dur.

---

## 6. Mouvement & animation

### 6.1 Durées (`motion.duration`)

| Token | Valeur | Usage |
|---|---|---|
| `instant` | `80ms` | Micro-retour (press) |
| `fast` | `140ms` | Hover, sorties |
| `base` | `220ms` | **Défaut** (entrées, transitions) |
| `slow` | `320ms` | Transitions complexes |
| `slower` | `480ms` | Séquences, stagger |

### 6.2 Courbes (`motion.easing`)

| Token | Courbe | Rôle |
|---|---|---|
| `standard` | `cubic-bezier(0.4, 0, 0.2, 1)` | Usage général |
| `organic` | `cubic-bezier(0.22, 1, 0.36, 1)` | **Signature** — entrées naturelles |
| `decelerate` | `cubic-bezier(0, 0, 0.2, 1)` | Entrée d'un élément |
| `accelerate` | `cubic-bezier(0.4, 0, 1, 1)` | Sortie d'un élément |
| `springSoft` | `cubic-bezier(0.34, 1.56, 0.64, 1)` | Rebond doux (indicateurs, toggles) |

### 6.3 Règles

- La brique d'animation avancée est **`motion` (motion/react, ex-Framer) v12**, **interne à CanopUI** — jamais ré-exportée aux portails.
- Toute animation passe par un token de durée + une courbe. Pas de valeurs libres.
- **`prefers-reduced-motion: reduce`** : le thème force `animation/transition-duration: 0.01ms` globalement et `scroll-behavior: auto`. Les composants animés doivent aussi proposer un rendu statique équivalent.

---

## 7. Décor — `ShapeBackground`

Fond décoratif signature : des **formes organiques vertes floutées** (cercles, triangles à angles arrondis) réparties et légèrement animées, qui donnent la sensation de « terrain vivant » sans nuire à la lisibilité.

- **Couleur** : dérivée de `primary` (`primary.light` / `primary.main`), très floutée et discrète.
- **Modes** : `hero` (présent, écrans d'auth / héros) et `ambient` (discret, pages denses — un peu plus présent vers le centre).
- **Mouvement** : moteur physique léger (dérive lente, collisions douces entre formes) piloté par `requestAnimationFrame` ; les triangles sont floutés comme les autres formes.
- **Reduced-motion** : rendu **statique** (aucune animation) lorsque l'utilisateur le demande.
- **Lisibilité d'abord** : le décor reste toujours en arrière-plan, contraste du contenu garanti.

---

## 8. Iconographie

- Composant `Icon` — SVG maison, variantes **`outline`** et **`solid`**, taillées (`xs`→`lg`).
- Recoloration par **`currentColor`** : une icône hérite de la couleur de son contexte (jamais de couleur en dur dans le SVG).
- Jeu d'icônes métier étendu (briefcase, shoppingCart, car, wallet, receipt, barChart, calendar, tag, gift, coffee, plane, star…) pour couvrir les catégories des portails.
- Les **favicons** des portails suivent la DA : carré arrondi vert `#1e6244` + glyphe **or** `#f4ad15` (accent rare, cohérence de marque).

---

## 9. Responsive

### 9.1 Breakpoints (`breakpoints`, unité **rem**)

| Palier | Seuil | Cible |
|---|---|---|
| `xs` | `0` | Mobile |
| `sm` | `37.5rem` (600 px) | Grand mobile / petite tablette |
| `md` | `64rem` (1024 px) | Tablette large / desktop |
| `lg` | `90rem` (1440 px) | Grand desktop |

> Le breakpoint MUI `xl` est **désactivé** (`BreakpointOverrides { xl: false }`) : l'échelle s'arrête à `lg`.

### 9.2 Trois paliers d'expérience

- **Mobile** (`< 37.5rem`) : navigation en **bottom-nav flottante**, panneaux en **bottom-sheet** (poignée + swipe-to-close), padding bas avec `env(safe-area-inset-bottom)`.
- **Tablette** (`37.5rem`–`64rem`) : navigation en **rail à icônes** (icons-only ; `title` et `sidebarWidget` masqués — choix DA).
- **Desktop** (`≥ 64rem`) : navigation en **sidebar** complète.

### 9.3 Règles responsive

- **Jamais de scroll horizontal** : le contenu large (tables, diagrammes) scrolle dans son propre conteneur.
- **Cibles tactiles** confortables sur mobile (`≥ 3rem`).
- **`PageContent`** borne la largeur de lecture (`maxWidth` : `sm`/`md`/`lg`/`full`, défaut `lg`) et centre la colonne ; gouttières responsive par palier.
- Les tableaux (`DataTable`) basculent en **cartes empilées** sous `md`.

---

## 10. Thème & CSS variables

Le thème est fourni par **`ChThemeProvider`** (ThemeProvider MUI + CssBaseline). `createChTheme(mode)` produit le thème pour `light` ou `dark`.

- **Préfixe des variables** : `cssVarPrefix: "ch"` → toutes les variables sont **`--ch-*`** (ex. `--ch-palette-primary-main`, `--ch-shadow-e3`, `--ch-motion-ease-organic`, `--ch-radius-lg`, `--ch-spacing-md`).
- Les composants résolvent leurs couleurs via `var(--ch-…)` **au runtime** : aucune valeur figée au rendu → **thème surchargeable sans rebuild** (dark mode, white-label : surcharger 2-3 variables dans une feuille chargée après la lib).
- **Modes** : `useChTheme()` renvoie `{ mode, resolvedMode, setMode, toggleMode }`. `mode` = préférence (`light`/`dark`/**`system`**), `resolvedMode` = mode effectif (`system` résolu via `prefers-color-scheme`). Préférence persistée dans `localStorage` (clé `ch-theme-mode`).
- **`ThemeToggle`** : bascule **2 états** (Light / Dark) avec ombre et micro-animation ; intégré à la navigation (plus flottant sur la page).

Extensions de thème notables : palette `accent` ajoutée à MUI (dispo comme `color="accent"` sur `Button`, `CircularProgress`, `LinearProgress`), `surface.{sunken,base,raised,overlay}`, `text.{onPrimary,onAccent}`. Les champs (`OutlinedInput`) ont un fond teinté `secondary.light` mélangé au papier et une bordure `primary` de 2 px.

---

## 11. Patterns de composants (extrait)

L'inventaire complet et interactif vit dans la **vitrine**. Points DA saillants :

- **`Button`** — variantes `primary` (vert, action) / `secondary` / `danger` ; micro-interaction au press (`scale`, `springSoft`) ; état `loading`.
- **`Card` / `CardGrid`** — surfaces à élévation tokenisée, variantes (`surface`, `floating`, `stat`), rayons `lg`/`xl` ; grille fluide `auto-fit`.
- **`StatCard`** — chiffre clé (`metric`), variation, icône, fond teinté.
- **`SegmentedControl`** — onglets exclusifs avec **indicateur glissant** (`SlidingIndicator`, `springSoft`).
- **`Donut` / `ProgressBar` (segmentée)** — visualisations animées en `scaleX` (reduced-motion respecté).
- **`Navbar`** — sidebar / rail / bottom-nav selon le palier, indicateur d'onglet actif glissant partagé.
- **`PageScaffold` / `PageContent`** — header structuré (titre / sous-titre / actions), backdrop `ShapeBackground`, entrée en fondu.
- **`SidePanel`** — drawer latéral desktop → bottom-sheet mobile (poignée, swipe-to-close), sur `Modal` (focus trap + `Escape`).
- **`DataTable`** — en-têtes sticky, effet de scroll, bascule en **cartes** sous `md`, colonnes `hideOnMobile`.
- **Feedback** — `Feedback` / `Toast` (`success`/`error`/`info`/`warning`), `StatusChip`, `EmptyState` (illustration `ShapeBackground` + message + action).

---

## 12. Accessibilité

- **Contraste** : les `contrastText` de chaque rôle garantissent la lisibilité ; texte secondaire/disabled calibrés par mode.
- **Focus visible** : contour `primary` de 2 px, `outline-offset`, sur tout élément interactif.
- **Reduced-motion** : neutralisation globale des animations + rendus statiques des décors.
- **Sémantique** : `Heading` sépare niveau (a11y) et taille (visuel) ; overlays (`SidePanel`, `Menu`) avec focus trap et fermeture clavier.
- **Cibles tactiles** ≥ `3rem` sur mobile.

---

## 13. Règles d'or (do & don't)

**À faire**
- Nommer un **rôle** de palette, un **token** d'espacement/rayon, une **durée/courbe** de motion.
- Vérifier chaque écran en **clair ET sombre**, aux **3 paliers**, et en **reduced-motion**.
- Passer l'action primaire en **vert** ; réserver l'**or** à l'exceptionnel.

**À ne pas faire**
- Écrire une couleur hex, un `px`, un `rgba()` ou un `ms` en dur dans un composant.
- Importer MUI directement depuis un portail (toujours passer par `canopui`).
- Ré-exporter `motion` aux portails.
- Introduire un easing linéaire ou un saut d'animation.
- Utiliser l'or comme couleur d'action ou en aplat massif.

---

## 14. Références

- **Tokens (source de vérité)** : `src/tokens/` du dépôt CanopUI — `palette.ts`, `typography.ts`, `spacing.ts`, `radius.ts`, `shadows.ts`, `motion.ts`, `breakpoints.ts`.
- **Thème** : `src/theme/createChTheme.ts` (mapping `--ch-*`, extensions MUI).
- **Vitrine** : [canopui.qvl-project.com](https://canopui.qvl-project.com) — démos, props, code.
- **Vue d'ensemble & API** : [README CanopUI](README.md).
- **Versioning / release** : voir le README (semver, publication auto sur tag `vX.Y.Z`).

---

_DA maintenue par la flotte QVL-Studio · appliquée à Authenticator, Admin, Drive et Budgy._
