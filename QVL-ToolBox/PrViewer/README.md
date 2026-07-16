# PrViewer

## Rôle

PrViewer est une web app qui rassemble tes Pull Requests de plusieurs outils sur un seul écran et ne fait remonter que celles qui ont **bougé depuis ta dernière visite** (commentaire, rejet, nouveaux commits). Elle est **agnostique de l'outil** : Azure DevOps est livré comme provider de référence, mais l'app est bâtie en **ports & adapters** pour brancher Jira, GitHub, GitLab, etc. en écrivant un seul fichier, sans toucher à l'UI.

## Stack

- React + TypeScript + Vite.
- Node.js 18+.
- Proxy CORS intégré à Vite (les API des outils n'autorisent pas les appels directs depuis le navigateur). Un build statique servi seul aurait besoin d'un proxy équivalent côté serveur.

## Port

Serveur de dev Vite sur `5180`.

## Fonctionnalités

- Onglet « Mes PR » (PR dont tu es l'auteur) et onglet « À reviewer » (PR où tu es reviewer), groupés par dépôt/projet.
- Suivi vu / pas-vu : une PR n'apparaît que si son activité est plus récente que la date à laquelle on l'a marquée vue. La consulter l'ouvre dans l'outil d'origine et la marque comme vue.
- Cartes colorées selon le type de la dernière activité.
- Rafraîchissement automatique configurable (défaut 2 min, `0` désactive).
- Échecs partiels non bloquants : si une source est inaccessible, les autres s'affichent avec un avertissement.

## Architecture (ports & adapters)

L'app ne connaît qu'un contrat (le port `PrProvider`) et un modèle de PR neutre (`PrView`). Tout le spécifique à un outil vit dans un adapter ; la dépendance pointe toujours vers le domaine.

| Élément | Emplacement | Rôle |
|---|---|---|
| Domaine neutre | `src/types.ts` | `PrView`, `UpdateKind`, `Role`, sans trace d'un outil. |
| Port | `src/providers/types.ts` | `PrProvider`, `FieldDef`, `FetchResult`. |
| Adapters | `src/providers/<outil>.ts` | Implémentent le port (référence : `ado.ts`). |
| Registre | `src/providers/index.ts` | `PROVIDERS[]`, `getProvider()`. |

## Brancher un nouveau provider

1. Écrire `src/providers/<outil>.ts` exportant un objet `PrProvider` : `id`, `name`, `configFields`, `validateConfig()`, `fetchPullRequests()`.
2. L'ajouter au tableau `PROVIDERS` dans `src/providers/index.ts` : il apparaît aussitôt dans le sélecteur « Outil ».
3. Si l'API cible refuse le cross-origin, ajouter une entrée de proxy dans `vite.config.ts` sur le modèle de `/ado` et préfixer les URLs d'appel de l'adapter.

### Le modèle neutre `PrView`

C'est le seul type que l'UI comprend ; l'adapter traduit les objets de l'outil vers lui.

| Champ | Obligatoire | Rôle |
|---|:---:|---|
| `id` | oui | Identifiant stable (clé de liste + clé du suivi vu/pas-vu). |
| `title` | oui | Titre affiché. |
| `author` | oui | Nom affiché de l'auteur. |
| `group` | oui | Clé de regroupement (dépôt, projet). |
| `webUrl` | oui | URL d'ouverture dans l'outil d'origine. |
| `lastActivity` | oui | Date ISO de la dernière activité pertinente (voir l'invariant). |
| `updateKind` | oui | Catégorie d'activité, détermine la couleur de la carte. |
| `ref` | non | Référence courte affichée (défaut : `id`). |
| `source` / `target` | non | Branche source → cible (outils git). |
| `isDraft` | non | Affiche un badge « draft ». |

`fetchPullRequests` renvoie un `FetchResult` : `creator` (PR dont l'utilisateur est l'auteur), `reviewer` (PR où il est reviewer), `warnings` (échecs partiels non bloquants).

### L'invariant `lastActivity`

Tout le suivi vu/pas-vu en dépend. L'adapter doit : porter la date de la dernière activité *pertinente* (pas la date de création par défaut) ; filtrer le bruit (ses propres actions, votes positifs, ajouts de reviewers, mises à jour de build) ; garantir la monotonie croissante par `id` (`lastActivity` ne doit jamais régresser d'un fetch à l'autre). Implémentation de référence : la fonction `analyzeActivity` de `src/providers/ado.ts`.

### `updateKind` (couleur de la carte)

Déclaré dans `src/types.ts`, couleurs dans `src/updateKind.ts`.

| `updateKind` | Sens | Couleur |
|---|---|---|
| `comment` | commentaire d'une autre personne | ambre |
| `rejected` | rejet / changement demandé | rouge |
| `commits` | nouveau code poussé par quelqu'un d'autre | bleu |
| `new` | jamais consulté, sans activité | vert |
| `updated` | mise à jour générique (outils non-git) | bleu |

## Stockage & sécurité

| Donnée | Emplacement | Persistance |
|---|---|---|
| Config non secrète (org, projets, intervalle) | `localStorage` | persiste entre sessions |
| Champs `type: 'password'` (PAT, token) | `sessionStorage` | effacé à la fermeture de l'onglet |
| Suivi vu/pas-vu | `localStorage` | persiste entre sessions |

Mode recommandé : sans secret stocké. Pour Azure DevOps, laisser le PAT vide et utiliser `az login` ; le proxy injecte un jeton Azure AD côté serveur, qui ne transite jamais par le bundle navigateur. Les secrets ne sont jamais réaffichés dans l'UI ni écrits dans les logs. Le schéma de config est versionné (`CONFIG_VERSION`).

## Scripts npm

| Script | Effet |
|---|---|
| `npm run dev` | Serveur de dev Vite + proxy (port 5180). |
| `npm run build` | `tsc -b` (typecheck strict) puis build de production dans `dist/`. |
| `npm run preview` | Sert le build de production localement. |

> Le guide d'ajout de provider vit aussi dans le `PROVIDERS.md` du dépôt source, qui renvoie au README d'origine.
