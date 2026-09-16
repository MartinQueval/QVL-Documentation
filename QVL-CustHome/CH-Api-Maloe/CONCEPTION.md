# CH-Api-Maloe — Document de conception

> **Statut : conception.** Aucun code n'existe. Ce document est le résultat d'une session de conceptualisation
> (2026-08-21) et sert de base au découpage en US. Il sera remplacé par un `README.md` de service
> classique le jour où l'implémentation démarre.

**Maloë** est l'IA auto-hébergée de l'écosystème CustHome : un moteur open source (Ollama) exécuté sur le
serveur QVL, exposé aux portails et à des clients Windows, doté d'une identité et d'une mémoire propres.

Le principe directeur : **le moteur est interchangeable, l'identité ne l'est pas.** Le modèle de langage
est un détail d'implémentation que l'on remplace au fil des sorties open weights ; Maloë — son nom, son
ton, sa mémoire, ses règles — est un artefact versionné, propriété du projet.

---

## 1. Périmètre et phases

Quatre phases, séquentielles, chacune livrable indépendamment. **La phase N ne démarre qu'une fois la
phase N-1 en usage quotidien réel** — le socle doit être validé par l'usage avant d'être élargi.

| Phase | Contenu | Prérequis matériel | Prérequis logiciel |
|---|---|---|---|
| **1 — Conversation** | Chat sans action. Socle complet : Core, registre de modèles, persona, mémoire, console admin, portail web. | Aucun (matériel actuel suffit) | — |
| **2 — Assistant de code** | Client Windows agentique (type Claude Code), moteur spécialisé dev, skills et agents. | **24 Go de VRAM minimum** | Boucle agentique en lib partagée |
| **3 — Accès aux API CustHome** | Maloë lit et agit dans Budgy, Drive… sur délégation explicite de l'utilisateur. | Confortable à partir de 24 Go | **Grant Maloë** dans l'Authenticator (§7) |
| **4 — Assistant personnel vocal** | Client Windows vocal, wake word, routines, apprentissage, pilotage de la machine. | Aucun côté serveur (STT/TTS côté client) | Phases 1 et 3 |

Une conséquence à assumer dès maintenant : **la phase 2 est bloquée par le matériel, sans contournement.**
Un modèle de code utilisable démarre autour de 30 milliards de paramètres. Rien ne permet de tricher.

---

## 2. Architecture cible

```
SERVEUR (Linux headless, dédié)

  CH-Api-Maloe (Rust / Axum)  :8185
    ├── ordonnanceur         1 slot, FIFO, annulation propre
    ├── registre de modèles  tiers, états, sonde matérielle, résidence VRAM
    ├── persona              versionnée, compilée en deux densités
    ├── mémoire              PostgreSQL + pgvector, cloisonnée par utilisateur
    ├── registre d'actions   typé, classé par risque
    └── scheduler            routines, briefings
              │
              ├──► Ollama (127.0.0.1, jamais exposé au réseau)
              ├──► PostgreSQL + pgvector
              ├──► CH-Relay (publication des flux et des événements)
              └──► CH-Api-Authenticator (validation des tokens, échange de grants)

CLIENTS

  CH-Portal-Maloe   :3204   React + CanopUI — chat, mémoire, consentements, console admin
  CH-Maloe-Code             Windows, phase 2 — outils exécutés en local
  CH-Maloe-Desk             Windows, phase 4 — wake word, STT, TTS en local
```

### Principe de répartition

**Le serveur pense, le client agit.** Aucun média ne quitte la machine du client : le code source reste
sur le poste de dev, l'audio reste sur le poste de l'utilisateur. Seul du texte circule.

| Composant | Où | Pourquoi |
|---|---|---|
| Inférence LLM | Serveur | Seul endroit disposant du GPU |
| Persona, mémoire, routines | Serveur | État partagé entre tous les clients |
| Exécution des outils de code | Poste de dev | Le code n'est pas sur le serveur |
| Wake word, STT, TTS | Poste utilisateur | Latence, et l'audio ne doit pas transiter |

### Stack

- **Langage** : Rust (edition 2024), Axum — cohérence avec `CH-Api-Authenticator`, `CH-Api-Drive`,
  `CH-Api-Budgy` et `CH-Relay`. Le Gateway est le seul service Go de CustHome ; Maloë ne fait pas
  exception à la règle Rust.
- **Stockage** : PostgreSQL + extension **pgvector**. Un seul datastore pour les conversations, la
  mémoire, le registre de modèles, les jobs de routines et les embeddings. Pas de vector store dédié,
  pas de Redis, pas de broker supplémentaire — le volume ne le justifie pas et chaque brique est un
  coût d'exploitation.
- **Moteur d'inférence** : Ollama, en écoute sur la loopback exclusivement.
- **Port** : `8185` (prochain libre après Missive `8184`).

---

## 3. Ordonnancement

L'inférence est **sérialisée** : un modèle traite une requête à la fois. Avec 5 à 6 utilisateurs et
rarement deux simultanés, l'ordonnanceur reste volontairement minimal :

- un **slot unique** protégé par un mutex, file **FIFO** — à deux concurrents maximum, le FIFO *est* l'équité ;
- **annulation propre** : déconnexion du client → la génération est tuée, le slot libéré ;
- **plafond de longueur de génération**, pour qu'un modèle qui boucle ne bloque pas le slot.

Pas de quotas par utilisateur, pas de rate limiting applicatif, pas de budget de tokens. Ces mécanismes
ne se justifieront que si Maloë est un jour ouverte plus largement.

> **Décision (2026-08-21)** — l'ordonnanceur est dimensionné pour 5-6 utilisateurs et 2 simultanés
> maximum. L'équité par utilisateur et les quotas sont explicitement hors périmètre. À revoir seulement
> si le nombre d'utilisateurs change d'ordre de grandeur.

Corollaire : **la montée en GPU se justifie par la taille de modèle, pas par la concurrence.** L'inférence
batchée multi-utilisateurs n'apporte rien à cette échelle.

---

## 4. Modèles

### Tiers

| Tier | Rôle | Ordre de grandeur | État initial |
|---|---|---|---|
| **Utilitaire** | Embeddings pour la mémoire | ~150 Mo (Q8) | Activé, résident en permanence |
| **Tier 0** | Conversation, voix, classification, narration de routines | 3 B, Q4 (~1,9 Go) | Activé |
| **Tier 1** | Raisonnement intermédiaire | 14 B | Désactivé |
| **Tier 2** | Plafond généraliste | 32 B, Q4 (~20 Go) | Désactivé |
| **Tier dev** | Assistant de code (phase 2) | 30-32 B spécialisé | Désactivé |

Le choix des modèles précis est délibérément absent : l'offre open weights bouge trop vite pour être
figée dans un document de conception. Ce qui est figé, c'est **la grille de tiers et le plafond à 32 B**.

> **Décision (2026-08-21)** — le plafond assumé est la bande **32 B**. Les modèles au-delà (70 B, MoE à
> plusieurs centaines de milliards) sont hors périmètre : ils demandent une infrastructure de baie, pas
> un serveur maison, et 32 B suffit à tous les usages décrits, phase 2 comprise.

### Mécanique d'activation

Chaque modèle est déclaré dans le registre avec ses exigences (`min_ram_gb`, `min_vram_gb`). Au démarrage,
le Core **sonde le matériel** et calcule un statut :

| Statut | Signification | UI |
|---|---|---|
| `active` | Chargé ou chargeable immédiatement | Bascule active |
| `available` | Matériel suffisant, modèle non téléchargé | Bascule active, déclenche le `pull` |
| `insufficient_hardware` | Matériel insuffisant | Bascule grisée, **avec la raison affichée** |

Activer un modèle `available` déclenche un `ollama pull` en tâche de fond, avec progression poussée sur
CH-Relay. Deux points à ne pas masquer dans l'UI :

- **« un clic » n'est pas instantané** : un modèle de 32 B représente ~20 Go de téléchargement.
- **le disque doit être le SSD**, impérativement. Les bascules de modèles sont fréquentes tant que la
  VRAM est contrainte ; depuis un disque mécanique, chaque chargement coûterait des minutes.

### Résidence VRAM

Le Core gère **explicitement** quels modèles sont résidents, avec éviction. Il ne délègue pas cette
responsabilité au `keep_alive` d'Ollama : sur 4 Go de VRAM, deux modèles résidents provoquent un OOM.

Cette complexité est **transitoire**. À 24 Go, le tier 0 et le tier 2 cohabitent presque ; à 48 Go, tout
tient. Le gestionnaire doit donc être simple et explicite, pas intelligent — sa tâche s'allègera avec le
matériel, elle ne se compliquera pas.

---

## 5. Matériel

### État actuel

| Composant | Spécification | Implication |
|---|---|---|
| CPU | Ryzen 5 3600X (6c/12t, Zen 2, AVX2) | Pas le moteur d'inférence. Dual-channel DDR4 ⇒ ~40 Go/s, soit 8-12 tok/s sur un 4 B en CPU pur. |
| GPU | GTX 1650 Super, **4 Go**, Turing sans tensor cores | **Le moteur réel.** ~196 Go/s, soit ~5× la bande passante CPU. |
| RAM | 16 Go DDR4 | Partagée avec tous les services CustHome. |
| Stockage | SSD 1 To + HDD 4 To | Modèles sur le SSD. HDD pour sauvegardes et logs. |
| OS | Windows → **migration Linux serveur prévue** | Voir ci-dessous. |

Ce qui tient dans 4 Go de VRAM, tout en GPU :

| Modèle (Q4) | Poids | Contexte atteignable | Débit estimé |
|---|---|---|---|
| 1,5-2 B | ~1,2 Go | large | 60-90 tok/s |
| **3 B** | **~1,9 Go** | **8-16 k** | **40-60 tok/s** |
| 4 B | ~2,5 Go | 4-8 k | 35-45 tok/s |
| 7-8 B | ~4,7 Go | ne tient pas — offload partiel | 8-12 tok/s |

**Le tier 0 est donc un 3 B intégralement en VRAM**, à 40-60 tok/s : plus rapide que la lecture humaine,
et dans le budget de latence de la phase 4.

### La migration Linux est un prérequis, pas un confort

Elle est chiffrable, et elle porte sur la ressource la plus rare :

- le bureau Windows réserve en permanence **300 à 500 Mo de VRAM** — soit **plus de 10 % d'une carte de
  4 Go**, consommés pour afficher un fond d'écran ;
- l'environnement graphique et ses services consomment **3 à 4 Go de RAM**, récupérables en headless.

> **Décision (2026-08-21)** — la migration du serveur vers Linux headless est un **prérequis de la phase 1**.
> Concevoir Maloë sur le matériel actuel sous Windows revient à la concevoir sur une plateforme transitoire
> tout en amputant d'un quart la ressource critique.

### Le vrai plafond n'est pas le modèle, c'est le cache KV

Calcul rarement fait avant achat. Un 32 B en Q4 pèse ~20 Go de poids. Le **cache KV** — la mémoire du
contexte — coûte de l'ordre de 256 Ko par token en FP16 pour un modèle de cette taille :

| Contexte | Cache KV (FP16) | Cache KV (Q8) | Total avec le modèle |
|---|---|---|---|
| 8 k | 2 Go | 1 Go | 21 Go |
| 32 k | 8 Go | 4 Go | **24 Go — à la limite** |
| 64 k | 16 Go | 8 Go | 28 Go — ne tient pas |

**24 Go de VRAM donnent un 32 B avec ~32 k de contexte**, à condition de quantiser le cache KV en Q8.
Suffisant pour les phases 1, 3 et 4 ; à l'étroit pour la phase 2, où le codage agentique consomme du
contexte massivement.

### Trajectoire d'upgrade

| Étape | Coût | Ce que ça débloque |
|---|---|---|
| **0 — Linux headless** | 0 € | ~400 Mo de VRAM, ~3 Go de RAM. Prérequis phase 1. |
| **1 — GPU 24 Go** | ~700 € (occasion) | Tier 1, tier 2, **phase 2 devient réelle** (contexte 32 k). |
| **1 bis — 2× 24 Go** | ~1400 € | 48 Go, tout est large. Exige 2 slots PCIe et une alim 1000 W+. |
| **1 ter — 32 Go en une carte** | ~2000 € | Phase 2 confortable, simplicité, silence. |
| **2 — Plateforme DDR5** | variable | Prévu. Sort l'AM4 (dual-channel DDR4, CPU de 2019) de l'équation. |

Sur la config DDR5 : **ne pas surpayer le CPU.** Dès que le modèle tient entièrement en VRAM, le GPU fait
tout le travail — un 6-8 cœurs milieu de gamme suffit. Le budget doit aller à la VRAM. 64 Go de RAM
système est un bon point : de quoi héberger tous les services CustHome, PostgreSQL et les buffers de
chargement sans y penser.

---

## 6. Identité de Maloë

### Nature de l'artefact

La persona n'est **pas** une chaîne en dur dans le code. C'est un fichier versionné, relu et modifié
comme du code, avec des couches distinctes :

| Couche | Contenu | Variable ? |
|---|---|---|
| **Identité** | Nom, neutralité, rapport à soi | Stable, jamais surchargée |
| **Registre** | Ton, niveau de langue, longueur par défaut, rapport à l'incertitude | Stable |
| **Lignes rouges** | Ce qu'elle ne fait pas, quoi qu'on lui demande | Stable |
| **Capacités** | Ce qu'elle peut réellement faire *ici et maintenant* | **Variable par phase et par client** |

Le bloc **Capacités** est ce qui empêche l'échec le plus courant des petits modèles : l'hallucination de
capacité — affirmer avoir fait une action qu'aucun outil ne permet.

### Neutralité

Maloë porte un nom neutre et une identité neutre. Concrètement, dans la persona :

- elle ne revendique aucun genre ;
- elle ne corrige pas l'utilisateur sur le genre ou les pronoms qu'il lui attribue — chacun l'identifie
  comme il veut ;
- elle ne fait pas de sa neutralité un sujet : elle ne l'explique que si on le lui demande.

### Contrainte technique : compiler la persona en deux densités

Un modèle de 3 B **suit mal un long prompt système**. Une persona de 1500 tokens sur un tier 0 en 8 k de
contexte, c'est de la dérive garantie et un quart du contexte consommé avant le premier message.

La persona doit donc être **compilée à deux niveaux de verbosité** depuis une source unique :

- **densité haute** — condensée, impérative, pour le tier 0 ;
- **densité basse** — développée, nuancée, pour les tiers 2 et dev.

Même identité, deux formulations. La source reste unique pour éviter la divergence.

### Contrat de sortie par client

Le registre varie selon le canal, sans que l'identité change :

| Client | Contrat |
|---|---|
| Portail web | Réponses structurées, markdown autorisé, longueur libre |
| Assistant de code | Concis, orienté action, pas de préambule |
| Assistant vocal | **Phrases courtes, prononçables, pas de markdown, pas de listes** |

En phase 4, la **voix** fait partie de l'identité. Choisir un timbre cohérent avec une identité neutre
est un travail de design à part entière, pas un paramètre technique.

### Évaluations de persona

Une petite suite de tests est **non négociable** si l'on veut réellement pouvoir changer de moteur sans
perdre Maloë : reste-t-elle neutre ? refuse-t-elle une action non accordée ? reste-t-elle brève en mode
vocal ? n'invente-t-elle pas de capacité ?

Sans ces évaluations, chaque changement de modèle est une roulette russe sur l'identité — et la promesse
« moteur interchangeable, identité stable » est fausse.

---

## 7. Mémoire

C'est la pièce la plus importante du projet, et voici pourquoi : **à petit modèle, la qualité de la
mémoire compte davantage que la qualité du modèle.** C'est là que se joue la différence entre un gadget
et un assistant.

Trois étages :

| Étage | Contenu | Stockage |
|---|---|---|
| **Faits et préférences** | Données structurées et durables sur l'utilisateur | Tables relationnelles |
| **Épisodique** | Résumés de conversations, indexés sémantiquement | pgvector |
| **Procédural** | Routines apprises : « quand je dis X, fais Y » | Tables relationnelles |

Deux exigences non négociables :

1. **Cloisonnement strict par utilisateur.** La clé est le `sub` du JWT CustHome. Aucune fuite possible
   entre utilisateurs, y compris via un cache d'embeddings mal scopé.
2. **Inspectable et éditable par l'utilisateur.** Le portail expose ce que Maloë sait, avec la
   possibilité de corriger et de supprimer. « Maloë, oublie ça » doit fonctionner.

> **Décision (2026-08-21)** — « Maloë apprend de l'utilisateur » signifie **un système de mémoire**, pas
> du fine-tuning. Le fine-tuning est écarté : trop lent, trop coûteux, et sujet à l'oubli catastrophique.
> Un éventuel LoRA plus tard servirait le **style**, jamais la connaissance.

---

## 8. Sécurité et délégation — le grant Maloë

### État de l'existant

Audit du code de `CH-Api-Authenticator` (2026-08-21) :

| Constat | Référence |
|---|---|
| JWT HS256, claims `sub`, `roles[]`, `ip`, `iss`, `aud[]`, `iat`, `exp` — **aucun `scope`** | `src/services/jwt.rs:15` |
| L'audience par service existe déjà (`drive → ch-api-drive`, `budgy → ch-api-budgy`) | `src/services/jwt.rs:123` |
| Mais **l'audience n'est pas vérifiée** : `validation.validate_aud = false` | `src/services/jwt.rs:102` |
| Le claim `ip` est contrôlé strictement contre `X-Client-IP` | `src/handlers/validate.rs:28` |
| Le claim `ip` n'est posé que pour les comptes `whitelist_only` | `src/handlers/login.rs:67` |
| Les sous-rôles existent dans le modèle (`RoleKind::{Portal, Sub}`) | `src/domain/role.rs:39` |
| `Portal` est un enum fermé à 4 variantes | `src/domain/role.rs:7` |
| Primitive de token opaque révocable disponible (32 octets, SHA-256 au repos) | `src/services/secure_token.rs` |
| Familles de refresh tokens avec rotation et détection de réutilisation | `src/handlers/session.rs` |

**Conclusion : il n'existe pas de tokens délégués ou scopés.** Si Maloë détient le token d'un utilisateur,
elle peut tout ce que cet utilisateur peut. Mais **trois des quatre briques nécessaires existent déjà** :
le mécanisme d'audience, la granularité des sous-rôles, et la primitive de token opaque révocable.

### Trois blocages concrets

1. **Le claim `ip` interdit la réutilisation d'un token utilisateur.** Maloë appelant Budgy depuis le
   serveur présenterait l'IP du serveur → rejet au `/validate`. Et une routine nocturne n'a aucune IP
   cliente. Actif uniquement pour les comptes `whitelist_only`, donc une mine plutôt qu'un mur — mais une
   mine que Maloë ne contrôle pas.
2. **Le TTL de 15 minutes est incompatible avec les routines.** Un briefing à 6 h doit lire le solde Budgy
   alors que l'access token de l'utilisateur est mort depuis des heures. La seule issue avec l'existant
   serait que Maloë détienne le **refresh token** : credential de 7 jours à pleins pouvoirs, *et* la
   rotation avec détection de réutilisation ferait qu'un refresh concurrent avec le navigateur
   invaliderait la famille — **l'utilisateur serait déconnecté**. Ce n'est plus un risque de sécurité,
   c'est un bug fonctionnel garanti.
3. **L'audience n'étant pas vérifiée**, la restriction par service que le token annonce n'est pas
   appliquée. C'est précisément le levier sur lequel Maloë doit s'appuyer.

### Le design retenu : le grant Maloë

Un nouveau type de credential dans l'Authenticator, réutilisant `secure_token` :

1. Depuis le portail Maloë, l'utilisateur **accorde explicitement** un accès : *« Maloë peut lire mon
   solde Budgy »*. Cela crée un **grant** — token opaque haché au repos, `user_id`, liste de scopes,
   expiration, **révocable à tout moment**.
2. Maloë présente le grant à l'Authenticator et reçoit un **JWT court, scopé, d'audience limitée au
   service cible, et sans claim `ip`** — délibérément, puisqu'elle appelle depuis le serveur.
3. Le service en aval **vérifie le scope**. C'est là qu'est le travail neuf.

Ce que ce design règle d'un seul coup :

- **Blocage 2 disparaît** — le grant est long, révocable et stocké côté serveur. Le briefing de 6 h
  échange son grant contre un JWT frais. Aucun refresh token détourné, aucune déconnexion parasite.
- **Blocage 1 disparaît** — le token de délégation ne porte pas d'`ip`.
- **L'injection de prompt devient inoffensive hors périmètre accordé.** Même si un libellé de transaction
  convainc un modèle de 3 B de tenter un virement, le token ne le permet pas. La protection est
  **structurelle** : elle ne repose pas sur le jugement du modèle. C'est essentiel, car un modèle de
  cette taille n'a aucune résistance à l'injection — ce n'est pas une hypothèse, c'est le mode de
  défaillance attendu.
- **L'écran de consentement vient gratuitement** — le portail liste les accès en cours avec un bouton
  révoquer par ligne.

> **Décision (2026-08-21)** — l'accès de Maloë aux API CustHome passe par un **grant explicite, scopé et
> révocable par utilisateur**. Le pattern `X-Internal-Secret` (`src/handlers/internal.rs:27`) est
> **explicitement exclu** pour Maloë : c'est un secret partagé à pleins pouvoirs, sans identité
> utilisateur — exactement le compte de service global à éviter.

### Chantier induit sur l'existant

| Travail | Où | Taille |
|---|---|---|
| Claim `scope` et émission de tokens scopés | Authenticator | Petit |
| Activer `validate_aud` | Authenticator + tous les services | Petit mais **transverse**, à faire prudemment |
| Stockage des grants et endpoints de consentement | Authenticator | Moyen |
| Vérification des scopes | Budgy d'abord, puis les autres | Moyen, par service |
| `Portal::Maloe`, route Gateway, topics Relay | 3 fichiers | Trivial |

Ce chantier est **borné et interne à CustHome**. Il ne s'agit pas de construire un serveur OAuth.

### Actions sur la machine (phase 4)

Maloë ne reçoit **jamais** « les droits admin et un terminal ». Elle reçoit un **registre d'actions
déclarées, typées et paramétrées** (`ouvrir_app`, `régler_volume`, `verrouiller_session`,
`lancer_routine`), chacune classée par risque :

| Classe | Comportement |
|---|---|
| `auto` | Exécutée directement |
| `confirm` | Exige une confirmation explicite de l'utilisateur |
| `interdit` | Refusée, quel que soit le phrasé |

Le shell libre est une capacité **à part, désactivée par défaut, et toujours en `confirm`**. La différence
entre un assistant et un lance-flammes tient exactement là : une commande mal entendue ne doit pas pouvoir
se traduire en suppression récursive.

### Routines et briefings

**Le modèle ne produit jamais les faits, il ne fait que les formuler.** Le solde, les tâches exécutées
pendant la nuit, la météo sont collectés de façon déterministe par le Core, qui passe un JSON au modèle
pour la seule mise en mots.

Sans cette règle, un briefing matinal inventera un solde bancaire. Avec un tier 0 de 3 B, ce n'est pas
une précaution théorique.

---

## 9. Transport : CH-Relay plutôt que SSE via le Gateway

Le broker MQTT 5.0 maison couvre déjà les besoins de Maloë :

- **auth JWT** avec le même secret et le même issuer que l'Authenticator (`iss = ch-api-authenticator`) ;
- **ACL par topic templatée sur les claims** (`drive/{sub}/#`), où *subscribe* doit être **subsumé** par
  un pattern autorisé — pas d'élargissement avec `#` ;
- MQTT-over-WebSocket : le même broker sert le navigateur, le client Windows natif et le mobile.

Deux conséquences :

- **Le cloisonnement multi-utilisateurs des flux est déjà résolu.** `maloe/{sub}/#` isole chaque
  utilisateur, garanti par le broker — pas par du code applicatif que Maloë devrait écrire correctement.
- **Le streaming de tokens ne doit pas passer en SSE par le Gateway.** Celui-ci applique un
  `timeout_seconds: 5` global via un `context.WithTimeout` qui tue la requête sans discussion
  (`CH-Api-GateWay/internal/proxy/proxy.go:95`) : une génération de 40 secondes meurt à 5. Publier sur le
  Relay contourne d'un coup le timeout, le buffering du reverse proxy et le rate limit de 10 req/s.

Pour la phase 4, MQTT est de surcroît le transport naturel : le briefing du matin, les tâches terminées
pendant la nuit, les notifications de routines sont du **push**, pas du polling.

> **Décision (2026-08-21)** — les flux de génération et les événements de Maloë passent par **CH-Relay**
> (MQTT 5.0), pas par du SSE proxifié. Les appels requête/réponse courts (historique, mémoire,
> consentements, administration) restent en HTTP via le Gateway, avec un `timeout_seconds` relevé sur la
> route `/api/maloe`.

---

## 10. Décisions de conception révisées en séance

Tracées parce qu'elles ont changé en cours d'analyse, et que le raisonnement importe autant que la conclusion :

- **Go → Rust pour le Core.** Go avait été retenu sur l'argument « le Core est de l'orchestration I/O ».
  L'argument ne tient pas face à la cohérence du dépôt : tout CustHome est en Rust sauf le Gateway, et
  CH-Relay — un service massivement concurrent et orienté I/O — est en Rust et fonctionne.
- **Ordonnanceur équitable → slot unique FIFO.** Dimensionné d'abord pour un multi-utilisateurs
  générique, puis réduit une fois la charge réelle connue (5-6 utilisateurs, 2 simultanés).
- **Le GPU ne se justifie pas par la concurrence.** L'inférence batchée avait été avancée comme argument
  d'upgrade ; elle n'apporte rien à cette échelle. Seule la taille de modèle compte.
- **SSE → MQTT pour les flux.** Choix révisé après découverte que CH-Relay couvrait déjà l'auth, l'ACL
  par utilisateur et le WebSocket.

---

## 11. Points ouverts

À trancher avant le découpage en US de la phase 1 :

1. **Cible d'upgrade GPU** — 24 Go d'occasion, 2× 24 Go, ou 32 Go en une carte. Détermine si la phase 2
   se fait à 32 k de contexte ou confortablement.
2. **Nommage des clients Windows** — `CH-Maloe-Code` et `CH-Maloe-Desk` sont des propositions ; la
   convention du dépôt ne couvre que `CH-Api-*` et `CH-Portal-*`.
3. **Ordre exact des chantiers de la phase 3** — l'activation de `validate_aud` est transverse à tous les
   services et mérite d'être traitée comme une US autonome, indépendamment de Maloë.
4. **Contenu détaillé de la persona** — la structure est définie (§6), le contenu est à écrire. C'est un
   travail d'auteur, pas d'architecte.
5. **Choix du TTS** — trouver un timbre réellement neutre est un chantier de design à ouvrir tôt, car il
   conditionne l'identité perçue de Maloë en phase 4.
