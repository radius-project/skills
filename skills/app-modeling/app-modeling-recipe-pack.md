# Default recipe pack

Reference for `app-modeling.md`. Explains how a Radius **recipe pack** turns the
developer-facing resource types used in `.radius/app.bicep` into real Azure infrastructure, and
provides a default Azure recipe pack covering the supported types.

This is platform-engineer material. Recipe packs are registered on the **environment**, not in
`app.bicep`; application authors only reference resource types. Modeling an application never
requires editing a recipe pack — but it explains where the `readOnly` properties an app consumes
(a database `host`, an object store `endpoint`) actually come from.

## Concepts

- **Recipe** — platform-specific IaC (a Bicep or Terraform module) that provisions the backing
  infrastructure for one resource type and maps results back onto the resource's `readOnly`
  properties.
- **Recipe pack** — a `Radius.Core/recipePacks` resource that bundles one recipe per resource
  type, so a platform engineer can register an entire catalog of backends at once.
- **Environment** — a `Radius.Core/environments` resource that references one or more recipe
  packs and supplies the cloud `providers` (subscription, resource group) the recipes deploy
  into. When an app resource is deployed, Radius looks up its type in the environment's recipe
  packs and runs the matching recipe.

## Recipe entry anatomy

Each entry in a pack's `recipes` map is keyed by the `<namespace>/<type>` it provisions:

```bicep
'Radius.Data/postgreSqlDatabases': {
  kind: 'bicep'                       // 'bicep' or 'terraform'
  source: 'mcr.microsoft.com/bicep/avm/res/db-for-postgre-sql/flexible-server:0.15.2'
  parameters: {                       // optional: values passed to the module
    name: '{{context.resource.properties.database}}'
    administratorLogin: '{{context.resource.secrets.dbsecret.username}}'
  }
  outputs: {                          // optional: module output -> resource property
    host: 'fqdn'                      // populates the resource's readOnly `host`
  }
}
```

- `kind` — `bicep` (module published to an OCI registry) or `terraform` (registry or git
  module).
- `source` — the module reference, version-pinned (`...:0.15.2`).
- `parameters` — values for the module. `{{context.*}}` expressions resolve per deployed
  resource: `context.resource.properties.<prop>` for developer-set inputs and
  `context.resource.secrets.<name>.<key>` for keys from a referenced `Radius.Security/secrets`.
- `outputs` — maps a module output name onto a resource property, populating the `readOnly`
  values the app consumes through connections.

## The default Azure recipe pack

One recipe pack mapping every supported resource type to a standard Azure Verified Module (AVM),
plus the supporting platform recipes for secrets and containers. Registering it on an
environment makes all of these types deployable on Azure.

```bicep
extension radius

@description('Azure subscription the environment provisions resources into.')
param azureSubscriptionId string

@description('Existing Azure resource group the environment provisions resources into.')
param azureResourceGroup string

resource defaultAzure 'Radius.Core/recipePacks@2025-08-01-preview' = {
  name: 'default-azure'
  properties: {
    recipes: {
      // Data — relational, cache, document, SQL
      'Radius.Data/postgreSqlDatabases': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/db-for-postgre-sql/flexible-server:0.15.2'
      }
      'Radius.Data/mySqlDatabases': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/db-for-my-sql/flexible-server:0.10.3'
      }
      'Radius.Data/redisCaches': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/cache/redis-enterprise:0.5.1'
      }
      'Radius.Data/mongoDatabases': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/document-db/database-account:0.19.0'
      }
      'Radius.Data/sqlDatabases': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/sql/server:0.21.4'
      }
      // Storage — blob / object
      'Radius.Storage/objectStorage': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1'
      }
      // AI — inference and search
      'Radius.AI/models': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/cognitive-services/account:0.15.0'
      }
      'Radius.AI/search': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/search/search-service:0.12.2'
      }
      // Messaging — streaming and broker
      'Radius.Messaging/kafka': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/event-hub/namespace:0.14.2'
      }
      'Radius.Messaging/rabbitMQ': {
        kind: 'bicep'
        source: 'mcr.microsoft.com/bicep/avm/res/service-bus/namespace:0.16.2'
      }
      // Supporting platform recipes
      'Radius.Security/secrets': {
        kind: 'bicep'
        source: 'ghcr.io/radius-project/kube-recipes/secrets:latest'
      }
      'Radius.Compute/containers': {
        kind: 'bicep'
        source: 'ghcr.io/radius-project/kube-recipes/containers:latest'
      }
    }
  }
}

resource env 'Radius.Core/environments@2025-08-01-preview' = {
  name: 'default'
  properties: {
    providers: {
      azure: {
        subscriptionId: azureSubscriptionId
        resourceGroupName: azureResourceGroup
      }
      // The secrets and containers recipes deploy Kubernetes resources, so the
      // environment needs a target namespace.
      kubernetes: {
        namespace: 'default'
      }
    }
    recipePacks: [
      defaultAzure.id
    ]
  }
}
```

### Type-to-recipe map

| Resource type | Azure backend | AVM recipe module |
|---|---|---|
| `Radius.Data/postgreSqlDatabases` | Azure Database for PostgreSQL | `avm/res/db-for-postgre-sql/flexible-server` |
| `Radius.Data/mySqlDatabases` | Azure Database for MySQL | `avm/res/db-for-my-sql/flexible-server` |
| `Radius.Data/redisCaches` | Azure Managed Redis | `avm/res/cache/redis-enterprise` |
| `Radius.Data/mongoDatabases` | Azure Cosmos DB (MongoDB API) | `avm/res/document-db/database-account` |
| `Radius.Data/sqlDatabases` | Azure SQL Database | `avm/res/sql/server` |
| `Radius.Storage/objectStorage` | Azure Blob Storage | `avm/res/storage/storage-account` |
| `Radius.AI/models` | Azure OpenAI | `avm/res/cognitive-services/account` |
| `Radius.AI/search` | Azure AI Search | `avm/res/search/search-service` |
| `Radius.Messaging/kafka` | Azure Event Hubs (Kafka surface) | `avm/res/event-hub/namespace` |
| `Radius.Messaging/rabbitMQ` | Azure Service Bus | `avm/res/service-bus/namespace` |
| `Radius.Security/secrets` | Kubernetes Secret (supporting) | `ghcr.io/radius-project/kube-recipes/secrets` |
| `Radius.Compute/containers` | Kubernetes workload (supporting) | `ghcr.io/radius-project/kube-recipes/containers` |

## Configuring a recipe

The pack above relies on module defaults. To shape what a recipe provisions — sizing,
credentials, the database or container to create — add `parameters` and `outputs` to an entry.
This PostgreSQL entry maps the developer-set `size` and `database` properties and the referenced
secret onto the AVM, and maps the server FQDN back onto the resource's `host`:

```bicep
'Radius.Data/postgreSqlDatabases': {
  kind: 'bicep'
  source: 'mcr.microsoft.com/bicep/avm/res/db-for-postgre-sql/flexible-server:0.15.2'
  parameters: {
    name: serverName
    administratorLogin: '{{context.resource.secrets.dbsecret.username}}'
    administratorLoginPassword: '{{context.resource.secrets.dbsecret.password}}'
    skuName: '{{context.resource.properties.size == "S" ? "Standard_B1ms" : "Standard_D2ds_v5"}}'
    databases: [
      { name: '{{context.resource.properties.database}}' }
    ]
  }
  outputs: {
    host: 'fqdn'
  }
}
```

Credentials never appear in the pack: the developer's `Radius.Security/secrets` resource holds
them, and the type's `secrets` binding exposes them to the recipe as
`context.resource.secrets.<name>.<key>`.

## Notes

- One recipe per type per pack. An environment can reference several packs (for example a
  datastore pack plus a compute pack) through the `recipePacks` array.
- `terraform` recipes use the same shape with `kind: 'terraform'` and a registry or git
  `source`.
- Pin module versions for reproducibility (`...:0.15.2`) rather than floating tags, except the
  supporting `ghcr.io/radius-project/kube-recipes/*` recipes, which track `latest` here.
- Recipe packs are an environment/platform concern. Application modeling
  (`app-modeling-radius-usage.md`) only references resource types — see the catalog in
  `app-modeling-available-resource-types.md`.
