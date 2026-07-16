# Missive

## Rôle

Missive est un mailer transactionnel générique et auto-hébergeable. Il expose une petite API HTTP et achemine un message vers un serveur SMTP, sans aucune logique métier : l'appelant fournit un sujet et un corps déjà rendus, Missive ne fait ni templating ni personnalisation. L'expéditeur est fixé côté serveur (`MAIL_FROM`) et n'est jamais dérivé du payload. Seul le canal `email` est opérationnel ; le canal `push` est réservé et répond `501`.

## Stack

- Rust (stable, édition 2021), version 0.1.0.
- Axum + Tokio pour le serveur HTTP, `lettre` pour le transport SMTP, `dotenvy` pour la configuration, `subtle` pour la comparaison de secret en temps constant.
- Logs au format JSON via `tracing`.

## Port

`127.0.0.1:8184` par défaut (bind loopback). Un bind non-loopback exige `ALLOW_EXTERNAL_BIND=true`, sinon le démarrage échoue.

## Configuration

Toute la configuration passe par variables d'environnement (ou fichier `.env`, voir `.env.example`). Une valeur vide ou blanche est traitée comme absente. Toute variable requise manquante ou invalide provoque un refus de démarrage immédiat (fail-fast).

| Variable | Rôle | Requis | Défaut |
|---|---|---|---|
| `PORT` | Port d'écoute HTTP (> 0). | Optionnel | `8184` |
| `BIND_ADDR` | Adresse d'écoute HTTP. | Optionnel | `127.0.0.1` |
| `ALLOW_EXTERNAL_BIND` | Autorise un bind non-loopback (toute valeur autre que `true` refuse). | Optionnel | `false` |
| `SMTP_HOST` | Hôte du serveur SMTP relais. | Requis | — |
| `SMTP_PORT` | Port SMTP. | Optionnel | `587` (avec credentials, STARTTLS) ou `25` (sans) |
| `SMTP_USER` | Identifiant SMTP (actif seulement si `SMTP_PASSWORD` est aussi présent). | Optionnel | — |
| `SMTP_PASSWORD` | Mot de passe SMTP (actif seulement si `SMTP_USER` est aussi présent). | Optionnel | — |
| `MAIL_FROM` | Expéditeur de tous les envois (format Mailbox). | Requis | — |
| `INTERNAL_API_SECRET` | Secret d'auth inter-services (header `x-internal-secret`), minimum 32 octets UTF-8. | Requis | — |
| `RATE_LIMIT_PER_MINUTE` | Envois max par minute (fenêtre fixe, global). Entier > 0. | Optionnel | `60` |
| `RUST_LOG` | Filtre de traces (format EnvFilter). | Optionnel | `info` |

Mode SMTP : `SMTP_USER` **et** `SMTP_PASSWORD` renseignés active l'authentification avec STARTTLS ; sinon connexion en clair sans TLS (réservée à un SMTP local de développement type Mailpit).

## Endpoints

| Méthode | Chemin | Auth | Description | Codes |
|---|---|---|---|---|
| GET | `/health` | Non | Sonde de disponibilité, renvoie `{"status":"ok"}`. | `200` |
| POST | `/v1/send` | `x-internal-secret` | Envoi d'un message. Le corps de requête est limité à 512 Ko. | `200`, `400`, `401`, `413`, `415`, `422`, `429`, `501`, `502`, `500` |

Corps de `POST /v1/send` :

```json
{
  "channel": "email",
  "to": "destinataire@example.com",
  "subject": "Sujet du message",
  "html": "<p>Corps HTML</p>",
  "text": "Corps texte"
}
```

Contraintes : `channel` vaut `email` (opérationnel) ou `push` (réservé, `501`) ; `to` est un destinataire unique et valide (une virgule est refusée) ; `subject` est requis et sans retour à la ligne ; `html` et `text` sont optionnels mais au moins l'un des deux doit être non vide (les deux produisent un message multipart).

Détail des codes de `POST /v1/send` :

| Code | Cause |
|---|---|
| `200` | Message accepté et transmis au serveur SMTP (`{"status":"sent"}`). |
| `400` | Payload invalide (canal inconnu, destinataire absent/invalide, plusieurs destinataires, sujet avec retour à la ligne, ni `html` ni `text`) ou JSON syntaxiquement invalide. |
| `401` | Header `x-internal-secret` absent, vide ou incorrect. |
| `413` | Corps au-delà de 512 Ko. |
| `415` | `content-type` absent ou différent de `application/json`. |
| `422` | JSON valide mais champ requis manquant ou de type incorrect. |
| `429` | Limite d'envois par minute atteinte. |
| `501` | Canal `push` : réservé, non implémenté. |
| `502` | Échec de transmission au serveur SMTP. |
| `500` | Erreur interne (message non constructible). |

## Format d'erreur

Les erreurs applicatives (`400` de validation, `401`, `429`, `501`, `502`, `500`) ont un **corps vide** : seul le code HTTP porte l'information. Les rejets faits par le framework avant d'atteindre la logique applicative (`400` JSON malformé, `413`, `415`, `422`) renvoient un corps `text/plain` décrivant le rejet.

## Contrat OpenAPI

Voir [openapi.yaml](./openapi.yaml).
