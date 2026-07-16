# QVL-ToolBox

Boîte à outils de la flotte QVL : services et briques réutilisables, conçus pour alimenter les projets QVL tout en restant génériques et auto-hébergeables. Les dépôts sources sont hébergés sur GitHub, organisation [github.com/QVL-ToolBox](https://github.com/QVL-ToolBox).

## Les outils

| Outil | Rôle | Documentation |
|---|---|---|
| AIGate | Passerelle IA compatible API OpenAI, une seule API pour OpenAI, Gemini, Claude et Mistral. | [AIGate](./AIGate/README.md) |
| Missive | Mailer transactionnel auto-hébergeable qui relaie un message vers un serveur SMTP. | [Missive](./Missive/README.md) |
| Relay | Broker de messages MQTT 5.0 auto-hébergé, bus de communication de la ToolBox. | [Relay](./Relay/README.md) |
| PipeBoard | Dashboard en lecture seule des pipelines gitlab.com, organisés par group et par repo. | [PipeBoard](./PipeBoard/README.md) |
| PrViewer | Visualiseur de Pull Requests multi-outils en ports & adapters (Azure DevOps de référence). | [PrViewer](./PrViewer/README.md) |
| HealthServ | Service de maintenance système : redémarrage planifié, purge RAM, relance de services. | [HealthServ](./HealthServ/README.md) |
| MasterEnv | Gestionnaire centralisé des variables d'environnement multi-repos avec détection de conflits. | [MasterEnv](./MasterEnv/README.md) |
| Switch | Orchestrateur de démarrage et d'arrêt ordonnés de la stack de services. | [Switch](./Switch/README.md) |

## Catégories

- APIs HTTP documentées avec contrat OpenAPI : AIGate, Missive.
- API HTTP interne : le backend Express de PipeBoard.
- Bus de messages standard (pas de REST) : Relay.
- Applications front / outillage local : PrViewer.
- Utilitaires système et orchestration (pas d'API HTTP) : HealthServ, MasterEnv, Switch.
