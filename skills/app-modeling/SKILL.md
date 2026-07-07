---
name: app-modeling
description: >
  Analyze a source code repository and generate a Radius application
  definition (.radius/app.bicep). Use when asked to create an application
  definition, model an application for Radius, or generate a Radius Bicep
  file. Resolves resource types from radius-project/resource-types-contrib
  and follows deterministic rules for validated output.
---

# Radius Application Modeling

Use this skill to generate a Radius application definition (`app.bicep`) from a source code repository.

## Response

When asked to model a repository, your chat reply should contain:

1. A one-line intro naming the app (e.g. "I'll create an application definition for `todo-list-app`.").
2. A short, natural summary of the application resources you identified — a brief list such as "Container: `todo-list-app`", "MySQL database", "Secret for DB credentials". A sentence or two of reasoning is fine; don't dump raw source analysis.
3. The `.radius/app.bicep` in a single code block.
4. An offer to open a pull request.

Don't create the pull request automatically — wait for the user to confirm. If they confirm, open a PR adding `.radius/app.bicep` with title `Add Radius application definition` and body `Add .radius/app.bicep for <app-name>.`

## Workflow

Before writing the Bicep:

1. Analyze the source (package manifest, Dockerfile/compose, entry point, persistence layer, env vars).
2. Classify into one architecture pattern — see [architecture-patterns.md](references/architecture-patterns.md).
3. Resolve the resource types the app needs from `radius-project/resource-types-contrib`. Derive each schema path from the type name (see [Resource Type Resolution](#resource-type-resolution)) and read only those files. If a needed type's schema isn't at the derived path, search the repo for `<typeName>.yaml`; if it still can't be resolved, stop and report the missing type rather than guessing. Types the app doesn't use don't need to resolve.
4. Apply the naming, structure, and secrets rules below (and in [bicep-structure-rules.md](references/bicep-structure-rules.md), [naming-conventions.md](references/naming-conventions.md), [secrets-handling.md](references/secrets-handling.md)).
5. Generate the Bicep and check it against the [validation checklist](#validation-checklist).

## Deterministic Naming Rules

These rules eliminate ambiguity. Apply them exactly.

### Symbolic names (left side of `=` in Bicep)

| Resource | Symbolic name |
|---|---|
| Application | `<shortName>App` where `<shortName>` is the app name without hyphens, camelCase (e.g., `todo-list-app` → `todoApp`) |
| Container | `<serviceName>Container` — service short name camelCase; single-container apps use `<shortName>Container` (e.g., `todoContainer`) |
| Container image | `<serviceName>Image` (e.g., `todoImage`) |
| Data store (database/cache/queue) | `<engine>` + role suffix, camelCase: `mysqlDb`, `postgresDb`, `neo4jDb`, `redisCache`. Multiple of the same engine: prefix with the source store name (e.g., `ordersPostgresDb`) |
| Data store secret | `<engine>Secret` (e.g., `mysqlSecret`) — one per data store that needs credentials |
| Route | `<serviceName>Route` (e.g., `todoRoute`) |

### Resource `name` properties (string values in Bicep)

| Resource | Name value |
|---|---|
| Application | Repository name in kebab-case (e.g., `'todo-list-app'`) |
| Container | Service name in kebab-case; single-container apps use the app name (e.g., `'todo-list-app'`) |
| Container image | `'<service-name>-image'` (e.g., `'todo-list-app-image'`) |
| Data store | Engine short name in kebab-case (`'mysql'`, `'postgres'`, `'neo4j'`, `'redis'`); multiple of the same engine use the source store name |
| Data store secret | `'<engine>-secret'` (e.g., `'mysql-secret'`) |

### Connection keys

| Connection | Key |
|---|---|
| Data store | Engine + role, lowercase: `mysqldb`, `postgresdb`, `neo4jdb`, `rediscache`. Multiple of the same engine: prefix with the source store name |
| Container image | `containerImage` (one built image per container; the key is scoped to that container's `connections` map) |

### Other fixed values

| Field | Value |
|---|---|
| Data store secret USERNAME | Derived from source DB config (e.g., `MYSQL_USER`, `POSTGRES_USER`, connection string); NEVER a superuser/admin account (`root`, `admin`, `sa`, `postgres`, `mysql`) — if the source uses one, fall back to `<shortName>_user` (e.g., `todo_user`) |
| Data store `database` name | Derived from source (e.g., `MYSQL_DATABASE`/`POSTGRES_DB`, or the database segment of a connection string) |
| Data store `version` | Derived from source (e.g., the image tag `mysql:8.0` → `'8.0'`) |
| Container key in `containers` map | Service short name camelCase (single-container: derived from app, e.g., `todo`) |
| Port key in `ports` map | `web` for the primary HTTP port; additional ports derive from protocol/use (`http`, `grpc`) |
| `build.context` for containerImages | Directory containing the Dockerfile, relative to repo root (`'.'` if at root) |

## Resource Type Resolution

### Built-in types (from `radius-project/radius`)

| Need | Resource Type | API Version |
|---|---|---|
| Application grouping | `Radius.Core/applications` | `2025-08-01-preview` |

`Radius.Core/applications` is built into the `radius` extension — there is no schema file for it in `resource-types-contrib`. Do NOT use `Applications.Core/applications` — the model has moved to `Radius.Core/applications`.

### Extensible types (from `radius-project/resource-types-contrib`)

Resolve each type's schema at runtime from the `radius-project/resource-types-contrib` repository. Do NOT hardcode a file path — derive it from the resource type name using the repo convention:

- Category = the segment after `Radius.` in the namespace (`Radius.Compute` → `Compute`, `Radius.Data` → `Data`, `Radius.Messaging` → `Messaging`, `Radius.AI` → `AI`, `Radius.Security` → `Security`)
- Schema path = `<Category>/<typeName>/<typeName>.yaml` (e.g., `Radius.Data/mySqlDatabases` → `Data/mySqlDatabases/mySqlDatabases.yaml`)

Read the schema file to get the exact property names, types, and API version. Use the API version declared in the schema (currently `2025-08-01-preview` for these types) rather than assuming a fixed value.

The following is the COMPLETE allow-list of types this skill may emit:

| Need | Resource Type |
|---|---|
| Container images (build from Dockerfile) | `Radius.Compute/containerImages` |
| Containers | `Radius.Compute/containers` |
| MySQL | `Radius.Data/mySqlDatabases` |
| PostgreSQL | `Radius.Data/postgreSqlDatabases` |
| Neo4j | `Radius.Data/neo4jDatabases` |
| MongoDB | `Radius.Data/mongoDatabases` |
| Redis (cache) | `Radius.Data/redisCaches` |
| SQL Server | `Radius.Data/sqlServerDatabases` |
| Kafka (event streaming) | `Radius.Messaging/kafka` |
| RabbitMQ (message queue) | `Radius.Messaging/rabbitMQ` |
| AI model endpoint | `Radius.AI/models` |
| AI search | `Radius.AI/search` |
| Persistent storage | `Radius.Compute/persistentVolumes` |
| External ingress | `Radius.Compute/routes` |
| Secrets | `Radius.Security/secrets` |

Do NOT use any type not listed above. Do NOT invent properties.

## Extension

Declare exactly one extension, `extension radius`. It provides every Radius type (`Radius.Core/*`, `Radius.Compute/*`, `Radius.Data/*`, `Radius.Messaging/*`, `Radius.AI/*`, `Radius.Security/*`). Do NOT declare per-namespace or per-type extensions (`radiusCompute`, `containers`, `kafka`, etc.).

## app.bicep Structure (mandatory order)

Declare resources in this order (do NOT output this as code — it is only for your reference):

1. Extension: `extension radius` (single, covers all Radius types)
2. Params: `environment`, then `@secure() password` if needed, then `@description(...) image` if needed
3. Application resource (`Radius.Core/applications@2025-08-01-preview`) — always exactly one
4. Data / infrastructure resources (databases, caches, message brokers, AI services)
5. Secret resources (database credentials, API keys)
6. Container image resources (if building from Dockerfile)
7. Container resources (with connections to images and infra)
8. Routes (only if external ingress needed)

Rules:
- One `Radius.Compute/containers` per container service; one `Radius.Data/*` per backing data store (engine/instance-derived symbolic name).
- Building from a Dockerfile: add a `Radius.Compute/containerImages` resource, pass the reference via `param image string`, and have the container use `<serviceName>Image.properties.image` with a connection to `<serviceName>Image.id`.
- Database credentials: create a `Radius.Security/secrets` resource and set `secretName` on the data store (password via `@secure() param`).
- Add `Radius.Compute/routes` only for external ingress.

## Connections

Wire containers to infrastructure via `connections`. Read [connection-conventions.md](references/connection-conventions.md) for the correct env var format.

Rules:
- NEVER duplicate auto-injected env vars with manual `env` entries.
- Only add explicit `env` entries for app-specific variables NOT covered by connection auto-injection.

## Secrets

See [secrets-handling.md](references/secrets-handling.md). Each data store with credentials references its secret via `secretName: <engine>Secret.name`. The password is a `@secure() param`; the username follows the naming rules above (never a superuser account).

## Bicep Structure Rules

Read [bicep-structure-rules.md](references/bicep-structure-rules.md) for all structural rules.

## Validation Checklist

Before returning the Bicep, verify:
- [ ] Exactly one `Radius.Core/applications@2025-08-01-preview`, and one `extension radius` (no per-namespace or per-type extensions).
- [ ] Every `Radius.*` type is on the allow-list and matches its schema; use the API version from the schema.
- [ ] `param environment string` is declared; `@secure() param password` and `param image string` are declared when needed.
- [ ] `connections` is a top-level object map under `properties` (not inside `containers`, not an array).
- [ ] Ports use `containerPort`; `build.context` is the Dockerfile's directory (`'.'` if at repo root).
- [ ] Data store `database`, `version`, and username are derived from source; the username is never a superuser account.
- [ ] No hardcoded passwords, no readOnly properties set, no comments in the Bicep, and no `bicepconfig.json`.

## Example

See [todo-list-app-example.md](references/todo-list-app-example.md) for a full worked example (`dockersamples/todo-list-app`).

Read [todo-list-app-example.md](references/todo-list-app-example.md) for a complete worked example. The generated Bicep in that example is the **expected correct output** for `dockersamples/todo-list-app`.