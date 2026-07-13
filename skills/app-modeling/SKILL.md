---
name: app-modeling
description: >
  Analyze a source code repository and generate a Radius application
  definition (.radius/app.bicep). Use when asked to create an application
  definition, model an application for Radius, or generate a Radius Bicep
  file. Resolves the configured Radius schemas and the application's runtime
  contract to produce validated, deployable output.
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

1. Inventory every executable workload and backing service from manifests, Dockerfiles, compose files, entrypoints, source configuration reads, and client initialization. Treat web, worker, migration, scheduler, and sidecar roles separately.
2. Extract a runtime contract for each workload: image/build context, entrypoint and arguments, listener and ports, required environment/configuration, secrets, writable storage, dependencies, and wire protocols. Follow [runtime-contract.md](references/runtime-contract.md).
3. Map backing services to Radius types with [component-catalog.md](references/component-catalog.md), using [architecture-patterns.md](references/architecture-patterns.md) only as context. Report unsupported essential components instead of substituting unrelated types.
4. Inspect the repository's `bicepconfig.json` and resolve every emitted type against the exact configured Radius extension. Use the matching `resource-types-contrib` schema revision and the target Environment's recipe/output contract when available; do not copy shapes from another version.
5. Choose a source build only when the repository has a complete, practical build context. Otherwise use a pinned published image. Follow [bicep-structure-rules.md](references/bicep-structure-rules.md).
6. Map every required runtime value using [connection-conventions.md](references/connection-conventions.md) and [secrets-handling.md](references/secrets-handling.md). A generic Radius connection is sufficient only when the source consumes the exact generic contract supplied by the configured version.
7. Generate the Bicep using [naming-conventions.md](references/naming-conventions.md), then compile it with the exact configured extension. Treat unknown type/property warnings as unresolved schema mismatches.
8. Perform the [validation checklist](#validation-checklist), including a static consistency pass against every workload's runtime contract. Compilation alone is not success.

## Deterministic Naming Rules

These rules eliminate ambiguity. Apply them exactly.

### Symbolic names (left side of `=` in Bicep)

| Resource | Symbolic name |
|---|---|
| Application | `<shortName>App` where `<shortName>` is the app name without hyphens, camelCase (e.g., `todo-list-app` → `todoApp`) |
| Container | `<serviceName>Container` — service short name camelCase; single-container apps use `<shortName>Container` (e.g., `todoContainer`) |
| Container image | `<serviceName>Image` (e.g., `todoImage`) |
| Data store (database/cache/queue) | `<engine>` + role suffix, camelCase: `mysqlDb`, `postgresDb`, `neo4jDb`, `redisCache`. Multiple of the same engine: prefix with the source store name (e.g., `ordersPostgresDb`) |
| Data store secret | `<engine>Secret` when the type requires `secretName`; `<engine>RuntimeSecret` when the app must consume a supplied credential via `secretKeyRef`; app secrets use `appSecrets` |
| Route | `<serviceName>Route` (e.g., `todoRoute`) |

### Resource `name` properties (string values in Bicep)

| Resource | Name value |
|---|---|
| Application | Repository name in kebab-case (e.g., `'todo-list-app'`) |
| Container | Service name in kebab-case; single-container apps use the app name (e.g., `'todo-list-app'`) |
| Container image | `'<service-name>-image'` (e.g., `'todo-list-app-image'`) |
| Data store | Engine short name in kebab-case (`'mysql'`, `'postgres'`, `'neo4j'`, `'redis'`); multiple of the same engine use the source store name |
| Data store secret | `'<engine>-secret'` or `'<engine>-runtime-secret'`; app secrets `'app-secrets'` |

### Connection keys

| Connection | Key |
|---|---|
| Data store | Engine + role, lowercase: `mysqldb`, `postgresdb`, `neo4jdb`, `rediscache`. Multiple of the same engine: prefix with the source store name |

### Other fixed values

| Field | Value |
|---|---|
| Data store admin username | The administrator username you author for the provisioned database. Set it wherever the schema puts credentials — `username` on the resource, or `USERNAME` in the secret when the schema uses `secretName`. Use a simple admin name (e.g., `myadmin`); it is NOT derived from the source |
| Data store `database` name | Derived from source (e.g., `MYSQL_DATABASE`/`POSTGRES_DB`, or the database segment of a connection string) |
| Data store `version` | Derived from source (e.g., the image tag `mysql:8.0` → `'8.0'`) |
| Container key in `containers` map | Service short name camelCase (single-container: derived from app, e.g., `todo`) |
| Port key in `ports` map | `web` for the primary HTTP port; additional ports derive from protocol/use (`http`, `grpc`) |
| `build.source` for containerImages | Repo git URL: `git::https://github.com/<org>/<repo>.git//<subdir>?ref=<sha-or-tag>` (`//<subdir>` only when the Dockerfile isn't at the repo root) |

## Resource Type Resolution

### Built-in types (from `radius-project/radius`)

| Need | Resource Type | API Version |
|---|---|---|
| Application grouping | `Radius.Core/applications` | `2025-08-01-preview` |

`Radius.Core/applications` is built into the `radius` extension — there is no schema file for it in `resource-types-contrib`. Do NOT use `Applications.Core/applications` — the model has moved to `Radius.Core/applications`.

### Extensible types (from `radius-project/resource-types-contrib`)

First inspect the target repository's `bicepconfig.json`: the `radius` extension alias is the compile-time contract. Resolve schemas from the `resource-types-contrib` revision that produced that artifact, or from the Environment's registered type definition. A mutable artifact such as `radius:latest`, a recipe tagged `latest`, or a branch ref can drift; warn about that uncertainty and do not mix its property shapes with a different revision.

Use the `radius-project/resource-types-contrib` repository for discovery. Do NOT hardcode a file path — derive it from the resource type name using the repo convention:

- Category = the segment after `Radius.` in the namespace (`Radius.Compute` → `Compute`, `Radius.Data` → `Data`, `Radius.Messaging` → `Messaging`, `Radius.AI` → `AI`, `Radius.Storage` → `Storage`, `Radius.Security` → `Security`)
- Schema path = `<Category>/<typeName>/<typeName>.yaml` (e.g., `Radius.Data/mySqlDatabases` → `Data/mySqlDatabases/mySqlDatabases.yaml`)

Read the matching schema file for property names, types, sensitivity, read-only outputs, and API versions. The configured extension and type registered in the target Environment must agree. Stop and report a version mismatch rather than choosing one contract or guessing.

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
| Object storage | `Radius.Storage/objectStorage` |
| Persistent storage | `Radius.Compute/persistentVolumes` |
| External ingress | `Radius.Compute/routes` |
| Secrets | `Radius.Security/secrets` |

Do NOT use any type not listed above. Do NOT invent properties.

## Extension

Declare exactly one extension, `extension radius`. It provides every Radius type (`Radius.Core/*`, `Radius.Compute/*`, `Radius.Data/*`, `Radius.Messaging/*`, `Radius.AI/*`, `Radius.Security/*`). Do NOT declare per-namespace or per-type extensions (`radiusCompute`, `containers`, `kafka`, etc.). The alias must resolve through the target repository's existing configuration; do not silently add or replace `bicepconfig.json`. Prefer an immutable extension reference when configuration is in scope.

## app.bicep Structure (mandatory order)

Declare resources in this order (do NOT output this as code — it is only for your reference):

1. Extension: `extension radius` (single, covers all Radius types)
2. Params: `environment`; add a `@secure() param` for each developer-supplied secret value
3. Application resource (`Radius.Core/applications@2025-08-01-preview`) — always exactly one
4. Data / infrastructure resources (databases, caches, message brokers, object storage, AI services)
5. Secret resources (app secrets, schema-required credentials, or runtime bindings for supplied secrets)
6. Container image resources (if building from Dockerfile)
7. Container resources (with image, configuration, secret, and dependency wiring)
8. Routes (only if external ingress needed)

Rules:
- One `Radius.Compute/containers` per container service; one `Radius.Data/*` per backing data store (engine/instance-derived symbolic name).
- Building from a complete Dockerfile context: add a `Radius.Compute/containerImages` resource with `build.source` set to the repo git URL (`git::https://github.com/<org>/<repo>.git//<subdir>?ref=<sha-or-tag>`); the container references the built image via `<serviceName>Image.properties.imageReference` (no separate connection needed).
- Database credentials follow the type's schema: if it defines `username`/`password`, set them on the resource; if it defines `secretName`, create a `Radius.Security/secrets` and reference it; if it defines neither, the type takes no credentials. Always use a `@secure() param` for the password.
- Add `Radius.Compute/routes` only for external ingress.
- Keep provider modules, SKUs, regions, firewall/network policy, and recipe output mapping in Environment/provider Bicep. `app.bicep` contains only developer intent and app-facing runtime wiring.

## Connections

`connections` declares a generic Radius relationship. It does not translate resource properties into arbitrary application-specific variable or configuration names. Read [connection-conventions.md](references/connection-conventions.md).

Rules:
- Inspect the source to identify the exact names, casing, value format, defaults, and configuration mechanism it consumes.
- Generic projection can be a `CONNECTION_<NAME>_PROPERTIES` JSON value, individual `CONNECTION_<NAME>_<PROPERTY>` values, or another version-specific shape. Verify the configured extension/runtime contract; do not assume one format.
- Use a connection alone only when the application explicitly consumes that applicable generic contract. Otherwise map each required native input explicitly from a verified nonsecret resource output, a secret reference, a literal/default, or runtime composition.
- A direct resource property or secret reference creates dependency ordering. Do not add a connection merely for ordering; retain one only when the application/tooling consumes the relationship.
- Explicit native variables may coexist with generic projection. Avoid conflicting values, and use `disableDefaultEnvVars` only when the exact container schema supports it and the generic variables would be harmful.

## Secrets

See [secrets-handling.md](references/secrets-handling.md). Secret contracts are version- and type-specific:

- **Inputs** (credentials you supply): follow the exact schema — sensitive resource properties, a referenced `Radius.Security/secrets`, or no credential input. Use `@secure()` for Bicep inputs, but remember it does not automatically protect every downstream `env.value` or interpolated property from state.
- **Outputs** (values a recipe generates): inspect the exact registered schema and recipe output mapping for the secret resource/path and key names. Prefer `valueFrom.secretKeyRef` when supported; do not assume every version exposes `<resource>.properties.secrets.name`.
- When an application requires a connection string containing a secret, bind the secret as a helper environment variable and compose at runtime with correct ordering and escaping. URL-encode credentials when the target syntax requires it.

## Bicep Structure Rules

Read [bicep-structure-rules.md](references/bicep-structure-rules.md) for all structural rules.

## Validation Checklist

Before returning the Bicep, verify:
- [ ] Exactly one `Radius.Core/applications@2025-08-01-preview`, and one `extension radius` (no per-namespace or per-type extensions).
- [ ] The file compiles with the target repository's exact configured extension; every `Radius.*` type is on the allow-list and matches that version's schema and API version. Unknown type/property warnings are resolved, not ignored.
- [ ] `param environment string` is declared; add a `@secure() param` for each developer-supplied secret.
- [ ] Every executable workload has the correct role, image/build, entrypoint/arguments, listener, exposed ports, environment/configuration, writable storage, and restart behavior. `containerPort` matches the process; it does not configure the listener.
- [ ] Every required app-native input is supplied with the exact name, casing, type, URL/config syntax, and intentional default found in source. Each declared generic connection is actually consumable by that source.
- [ ] A source build has a complete practical Dockerfile/context and uses an immutable ref; otherwise the published image is pinned. Generated builds are consumed through `.properties.imageReference`.
- [ ] Credentials match the type's schema: `username`+`password` on the resource, or `secretName`+secret, or none — whichever the schema defines. Password via `@secure() param`; `database`/`topic`/`queue`/etc. derived from source.
- [ ] Every secret follows the exact schema/recipe contract and uses `secretKeyRef` where supported. No secret is hardcoded or accidentally moved into plain state; runtime-composed values preserve ordering, escaping, and required encoding.
- [ ] Read-only properties are never **set**. A referenced nonsecret output exists in the exact schema and recipe; a referenced secret path/key exists in the exact secret-output contract. Such direct references provide dependency ordering.
- [ ] Provider behavior matches the client: FQDN, TLS, port, auth mode, protocol/version, connection-string syntax, and network access are verified. Provider modules, SKUs, regions, and firewall configuration remain outside `app.bicep`.
- [ ] A health endpoint is not treated as proof of dependency connectivity; perform the static consistency pass in [runtime-contract.md](references/runtime-contract.md).
- [ ] The generated Bicep contains no explanatory comments and does not add `bicepconfig.json`.

## Example

See [todo-list-app-example.md](references/todo-list-app-example.md) for a complete worked example whose source expects native database variables rather than Radius generic connection variables.