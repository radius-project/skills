# Available resource types

Reference for `app-modeling.md` and `app-modeling-radius-usage.md`. A catalog of the
developer-facing Radius resource types used to model an application's infrastructure
dependencies.

All types use the API version `2025-08-01-preview`, e.g.
`Radius.Data/postgreSqlDatabases@2025-08-01-preview`.

## Catalog

| # | Namespace | Type | Concept | Azure backend |
|---|---|---|---|---|
| 1 | `Radius.Data` | `postgreSqlDatabases` | PostgreSQL relational DB | Azure Database for PostgreSQL |
| 2 | `Radius.Data` | `mySqlDatabases` | MySQL relational DB | Azure Database for MySQL |
| 3 | `Radius.Data` | `redisCaches` | In-memory cache | Azure Managed Redis |
| 4 | `Radius.Data` | `mongoDatabases` | Document DB (JSON/BSON) | Azure Cosmos DB (MongoDB API) |
| 5 | `Radius.Data` | `sqlDatabases` | SQL Server (T-SQL) | Azure SQL Database |
| 6 | `Radius.Storage` | `objectStorage` | Blob / object storage | Azure Blob Storage |
| 7 | `Radius.AI` | `models` | Hosted LLM inference | Azure OpenAI |
| 8 | `Radius.AI` | `search` | Full-text search & analytics | Azure AI Search |
| 9 | `Radius.Messaging` | `kafka` | Event streaming | Azure Event Hubs (Kafka surface) |
| 10 | `Radius.Messaging` | `rabbitMQ` | Message broker (AMQP) | Azure Service Bus |

## Recipes

Each type is provisioned on its Azure backend by a recipe registered on the environment. See
`app-modeling-recipe-pack.md` for a default Azure recipe pack that maps every type above to a
concrete Azure module.
