# YogaTools MCP Server

> Derniere mise a jour : 2026-04-05

## Vue d'ensemble

YogaTools expose un serveur MCP (Model Context Protocol) pour permettre a un client compatible MCP d'interroger et d'agir sur les donnees metier d'un compte YogaTools.

Le serveur est expose en **Streamable HTTP** sur :

- `https://<votre-domaine>/api/mcp`
- en local : `http://localhost:5000/api/mcp`

Le scope actuel couvre **40 tools** repartis sur :

- clients
- portail client
- rendez-vous
- factures
- email
- workshops
- disponibilites
- vacances
- reductions

Le serveur expose actuellement **des tools uniquement**. Il n'expose pas de resources ni de prompts.

## Cas d'usage

Avec ce serveur MCP, un agent compatible peut par exemple :

- retrouver un client par nom, email ou telephone
- lire la fiche detaillee d'un client
- creer ou modifier un client
- activer l'acces portail d'un client
- lister, creer, confirmer, deplacer ou annuler un rendez-vous
- lire et envoyer une facture
- envoyer un email direct
- creer, publier ou archiver un workshop
- gerer les disponibilites et vacances d'un coach
- verifier ou creer un code promo

## Prerequis

Pour utiliser le serveur MCP YogaTools, il faut :

- un backend YogaTools en cours d'execution
- `@modelcontextprotocol/sdk` installe dans l'environnement runtime
- le module MCP active
- un token MCP valide

Variables d'environnement cote serveur :

```env
YOGATOOLS_MCP_ENABLED=true
YOGATOOLS_MCP_MODE=enabled

# Format recommande
YOGATOOLS_MCP_CALLERS=[{"name":"muneo","token":"<token>","accounts":["acc_123"]}]

# Fallback legacy
YOGATOOLS_MCP_INTERNAL_TOKEN=
```

Notes :

- `YOGATOOLS_MCP_MODE=shadow` active les controles mais n'execute pas les tools d'ecriture.
- `YOGATOOLS_MCP_MODE=disabled` desactive le serveur.
- `YOGATOOLS_MCP_CALLERS` permet de scoper chaque caller a une liste de comptes.
- en production, privilegiez des `accountId` explicites plutot qu'un scope global `["*"]`.

## Authentification

Toutes les requetes vers `/api/mcp` doivent contenir :

- `Authorization: Bearer <token>`
- `X-MCP-Internal: true`

Si vous appelez directement l'endpoint HTTP, ajoutez aussi :

- `Content-Type: application/json`
- `Accept: text/event-stream, application/json`

En cas d'erreur :

- `401` : header manquant ou token invalide
- `405` : tentative de fermeture de session en mode stateless
- `503` : SDK MCP indisponible sur cette instance

## Exemple TypeScript

Exemple avec le SDK officiel MCP JavaScript :

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("https://api.yogatools.app/api/mcp"),
  {
    requestInit: {
      headers: {
        Authorization: `Bearer ${process.env.YOGATOOLS_MCP_TOKEN}`,
        "X-MCP-Internal": "true",
      },
    },
  },
);

const client = new Client(
  { name: "my-mcp-client", version: "1.0.0" },
  { capabilities: {} },
);

await client.connect(transport);

const tools = await client.listTools();
console.log(tools.tools.map((tool) => tool.name));

const result = await client.callTool({
  name: "yogatools.list_clients",
  arguments: {
    accountId: "acc_123",
    limit: 20,
  },
});

const textBlock = result.content.find((item) => item.type === "text");
console.log(textBlock?.text);

await transport.close();
```

## Exemple HTTP brut

Pour du debug manuel, vous pouvez initialiser le serveur avec `curl` :

```bash
curl -X POST http://localhost:5000/api/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream, application/json" \
  -H "X-MCP-Internal: true" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "clientInfo": { "name": "manual-test", "version": "1.0.0" },
      "protocolVersion": "2024-11-05"
    }
  }'
```

Pour un usage applicatif normal, il est recommande d'utiliser un client MCP plutot que de gerer le protocole JSON-RPC a la main.

## Regles d'utilisation des tools

La plupart des tools YogaTools suivent ces regles :

- `accountId` est obligatoire
- les tools sensibles peuvent exiger `confirm: true`
- certaines ecritures exigent `idempotencyKey`
- les resultats sont sanitizes avant retour

Exemples :

- `yogatools.cancel_booking` exige `confirm: true`
- `yogatools.create_client` exige `idempotencyKey`
- `yogatools.create_booking` exige `idempotencyKey`
- `yogatools.deactivate_discount` exige `confirm: true`

## Catalogue des tools

### Clients

- `yogatools.list_clients`
- `yogatools.search_clients`
- `yogatools.get_client_profile`
- `yogatools.create_client`
- `yogatools.update_client`

### Portail client

- `yogatools.check_client_portal_access`
- `yogatools.enable_client_portal_access`
- `yogatools.send_client_portal_credentials`

### Rendez-vous

- `yogatools.list_appointments`
- `yogatools.get_appointment_details`
- `yogatools.create_booking`
- `yogatools.update_booking_details`
- `yogatools.reschedule_booking`
- `yogatools.cancel_booking`
- `yogatools.confirm_booking`

### Factures

- `yogatools.list_invoices`
- `yogatools.get_invoice_details`
- `yogatools.send_invoice`

### Email

- `yogatools.send_email`

### Workshops

- `yogatools.list_workshops`
- `yogatools.get_workshop_details`
- `yogatools.create_workshop`
- `yogatools.update_workshop`
- `yogatools.publish_workshop`
- `yogatools.archive_workshop`

### Disponibilites

- `yogatools.list_availabilities`
- `yogatools.create_availability`
- `yogatools.update_availability`
- `yogatools.delete_availability`

### Vacances

- `yogatools.list_vacations`
- `yogatools.create_vacation`
- `yogatools.update_vacation`
- `yogatools.delete_vacation`

### Reductions

- `yogatools.list_discounts`
- `yogatools.get_discount_details`
- `yogatools.find_discount_by_code`
- `yogatools.check_discount_validity`
- `yogatools.create_discount`
- `yogatools.update_discount`
- `yogatools.deactivate_discount`

## Securite et garde-fous

Le serveur MCP YogaTools n'est pas un acces direct a la base. Il applique :

- authentification par token de service
- resolution du caller et scope par compte
- isolation multi-tenant via `accountId`
- verification billing avant execution
- confirmation explicite sur certaines actions
- idempotence sur certaines ecritures
- sanitization des reponses
- audit des executions

## Compatibilite client

Le serveur est adapte aux clients MCP capables de parler **Streamable HTTP**, par exemple :

- Muneo
- agents custom Node.js
- tout client base sur le SDK officiel MCP

Si votre client ne supporte pas les headers custom ou le transport Streamable HTTP, il faudra ajouter une petite couche d'adaptation.

## Conseils d'integration

- commencez par `listTools()` pour decouvrir les tools disponibles
- utilisez un caller scope a un ou plusieurs `accountId` precis plutot qu'un token global
- envoyez `confirm: true` uniquement sur les actions que vous voulez vraiment executer
- generez un `idempotencyKey` unique pour chaque creation ou envoi sensible
- parsez le contenu texte JSON retourne par `tools/call`

## Limites actuelles

- pas de resources MCP
- pas de prompts MCP
- pas de WebSocket ni stdio dans cette implementation
- le serveur est pense pour des integrations service-to-service securisees

## Support

Si vous integrez le serveur MCP YogaTools dans un client ou un agent, les points les plus frequents a verifier sont :

- token MCP invalide ou absent
- `X-MCP-Internal: true` manquant
- module MCP non active
- `accountId` manquant dans les arguments
- `confirm: true` ou `idempotencyKey` oublies sur certains tools
