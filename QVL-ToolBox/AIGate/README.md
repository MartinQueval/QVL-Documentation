# AIGate

## Rôle

AIGate est une passerelle IA qui s'intercale entre les applications et les fournisseurs de modèles. Les applications parlent le format de l'API OpenAI ; AIGate traduit chaque requête vers le format natif du moteur ciblé (OpenAI, Gemini, Claude ou Mistral) et normalise la réponse. Un seul démon, une seule API compatible OpenAI pour tous les moteurs. Le moteur est choisi par un préfixe `provider/model` (par exemple `gemini/gemini-2.0-flash`) ou par l'en-tête `X-AI-Provider`.

Les clés d'API sont transmises par requête et ne sont jamais stockées. La clé du fournisseur voyage dans `Authorization: Bearer`, ou par moteur via `X-AI-Key-<provider>`.

## Stack

- Rust (édition 2021), version 0.1.0.
- Serveur HTTP Axum + Tokio, deux crates : `aigate-core` (trait `Provider`, types unifiés, adaptateurs) et `aigate-server` (démon HTTP).
- Moteurs supportés (ordre canonique) : `openai`, `mistral`, `claude` (alias `anthropic`), `gemini` (alias `google`).

## Port

`0.0.0.0:8080` (fixe dans le code).

## Configuration

Toutes les variables d'environnement sont optionnelles.

| Variable | Défaut | Rôle |
|---|---|---|
| `AIGATE_KEYS` | absent (auth désactivée) | Paires `clé:app` séparées par des virgules. Active l'authentification AIGate. |
| `AIGATE_RATE_LIMIT` | `0` (désactivé) | Requêtes/minute par identité (token bucket). |
| `AIGATE_CACHE_MAX` | `1000` | Nombre maximal d'entrées de cache (`0` = illimité, éviction LRU). |
| `AIGATE_STATE_FILE` | `aigate-state.json` | Chemin de persistance (`off`/`none`/vide désactive). |
| `AIGATE_FLUSH_SECS` | `15` | Intervalle de flush de la persistance (secondes). |
| `RUST_LOG` | `aigate_server=info,tower_http=info` | Filtre de traces. |

En-têtes de requête notables : `Authorization: Bearer <clé fournisseur>`, `X-AI-Key-<provider>` (clé par moteur, requis pour le failover cross-moteur), `X-AIGate-Key` (clé AIGate si l'auth est active), `X-AI-Provider`, `X-AI-App` (label pour les métriques), `X-AI-Cache` (`on` | `<secondes>` | `off`), `X-AI-Retries` (1-10). Sur les réponses non streamées, `X-AI-Cache: HIT|MISS` est renvoyé.

## Endpoints

| Méthode | Chemin | Auth | Description | Codes |
|---|---|---|---|---|
| GET | `/health` | Non | Sonde de disponibilité, renvoie `ok`. | `200` |
| GET | `/v1/models` | Oui* | Liste des modèles agrégée par moteur (ids préfixés `provider/model`). Filtre optionnel `?provider=<nom>`. Liste live si une clé est disponible, sinon catalogue intégré. | `200`, `400`, `401`, `429` |
| GET | `/v1/usage` | Oui* | Métriques de tokens et de coût agrégées depuis le démarrage, plus les statistiques de cache. | `200`, `401`, `429` |
| POST | `/v1/chat/completions` | Oui* | Complétion de chat (streaming ou non, outils, images, failover, cache). Corps au schéma OpenAI chat-completions plus l'extension `fallbacks`. | `200`, `400`, `401`, `429`, `502` |

\* L'authentification n'est exigée que lorsque `AIGATE_KEYS` est défini ; sinon toutes les routes `/v1/*` sont ouvertes. `/health` est toujours ouvert.

Précisions sur les codes de `/v1/chat/completions` :

- `400` : corps invalide, aucune cible utilisable, ou chaîne interrompue (erreur client non rejouable, par exemple `400`/`422` renvoyé par le moteur).
- `429` : limite de requêtes par minute atteinte (avec indice `retry_after` en secondes).
- `502` : tous les moteurs de la chaîne ont échoué (avec le détail des tentatives).

## Format d'erreur

Toutes les erreurs sont du JSON de forme :

```json
{ "error": { "message": "description" } }
```

Pour un échec de chaîne (`/v1/chat/completions`), le champ `attempts` détaille chaque tentative :

```json
{
  "error": {
    "message": "all providers failed",
    "attempts": [
      { "provider": "openai", "model": "gpt-4o-mini", "tries": 3, "error": "..." }
    ]
  }
}
```

Le `429` porte un `retry_after` :

```json
{ "error": { "message": "rate limit exceeded", "retry_after": 12 } }
```

## Contrat OpenAPI

Voir [openapi.yaml](./openapi.yaml).
