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

## Output Format

Your entire visible response must follow this exact sequence. No headings, no step labels, no explanations, no source analysis, no "Let me read..." preamble.

1. Say: I will create an application definition for `<app-name>`.
2. Say: First, let me review the platform constitution.
3. Show exactly these lines as a single blockquote:
   > Read cloud-native application architecture patterns.
   > Read available resource types.
   > Read resource types composition rules.
   > Read resource types dependencies.
   > Read naming conventions.
   > Read secret handling requirements.
4. Say: I see this repository has a single application called `<app-name>`.
5. Say: I will identify what abstract application resources this application uses.
6. Say: It has these application resources:
7. List resources as a numbered list (e.g. "1. Container: todo-list-app")
8. Say: An application definition has been created for `<app-name>`.
9. Output the `.radius/app.bicep` code block.
10. Say: Would you like me to create a pull request with this file?

That is the COMPLETE chat response. Nothing before step 1, nothing after step 10. Do NOT automatically create a pull request. Wait for the user to confirm.

If the user confirms, create a pull request adding `.radius/app.bicep` to the repository. The PR title should be `Add Radius application definition`. The PR body should say `Add .radius/app.bicep for <app-name>.` — nothing else.

## Internal Workflow (do NOT show these steps to the user)

Internally, before producing the output above:

1. Analyze the source repository (package manifest, Dockerfile/compose, entry point, persistence layer, env vars).
2. Classify into exactly one architecture pattern. Read [architecture-patterns.md](references/architecture-patterns.md).
3. Resolve resource types from `radius-project/resource-types-contrib` — MUST match existing schemas. Read only the relevant YAML schema files.
4. Read [bicep-structure-rules.md](references/bicep-structure-rules.md) for correct Bicep structure.
5. Read [naming-conventions.md](references/naming-conventions.md) and apply the Deterministic Naming Rules below.
6. Read [secrets-handling.md](references/secrets-handling.md).
7. Generate the Bicep and validate against the checklist.

Then produce the output in the exact format above. Do NOT output anything while performing these steps.

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
| Data store secret USERNAME | Derived from source DB config (e.g., `MYSQL_USER`, `POSTGRES_USER`, connection string); fallback `<shortName>_user` (e.g., `todo_user`) |
| Data store `database` name | Derived from source (e.g., `MYSQL_DATABASE`/`POSTGRES_DB`, or the database segment of a connection string) |
| Data store `version` | Derived from source (e.g., the image tag `mysql:8.0` → `'8.0'`) |
| Container key in `containers` map | Service short name camelCase (single-container: derived from app, e.g., `todo`) |
| Port key in `ports` map | `web` for the primary HTTP port; additional ports derive from protocol/use (`http`, `grpc`) |
| `build.context` for containerImages | Directory containing the Dockerfile, relative to repo root (`'.'` if at root) |

### Extension order

Always declare extensions in this exact order:
1. `extension radius`
2. `extension radiusCompute`
3. `extension radiusSecurity`
4. `extension radiusData`

## Resource Type Resolution

### Built-in types (from `radius-project/radius`)

| Need | Resource Type | API Version |
|---|---|---|
| Application grouping | `Applications.Core/applications` | `2023-10-01-preview` |

`Applications.Core/applications` is built into Radius. It uses `extension radius` — no additional extension needed. Its API version is `2023-10-01-preview`. Do NOT use `Radius.Core/applications` — it does not exist.

### Extensible types (from `radius-project/resource-types-contrib`)

Read the resource type YAML schema files from the `radius-project/resource-types-contrib` repository. Each resource type has a YAML file at `<Category>/<typeName>/<typeName>.yaml`.

| Need | Resource Type | Schema file in `resource-types-contrib` |
|---|---|---|
| Container images (build from Dockerfile) | `Radius.Compute/containerImages` | `Compute/containerImages/containerImages.yaml` (PR #126 — read from `willdavsmith:containerimages-v2` branch until merged) |
| Containers | `Radius.Compute/containers` | `Compute/containers/containers.yaml` |
| MySQL | `Radius.Data/mySqlDatabases` | `Data/mySqlDatabases/mySqlDatabases.yaml` |
| PostgreSQL | `Radius.Data/postgreSqlDatabases` | `Data/postgreSqlDatabases/postgreSqlDatabases.yaml` |
| Neo4j | `Radius.Data/neo4jDatabases` | `Data/neo4jDatabases/neo4jDatabases.yaml` |
| Persistent storage | `Radius.Compute/persistentVolumes` | `Compute/persistentVolumes/persistentVolumes.yaml` |
| External ingress | `Radius.Compute/routes` | `Compute/routes/routes.yaml` |
| Secrets | `Radius.Security/secrets` | `Security/secrets/secrets.yaml` |

This is the COMPLETE list. Do NOT use any type not listed above. Do NOT invent properties. All extensible types use API version `2025-08-01-preview`.

## Extension naming

Bicep extensions are named by namespace, NOT by individual type:

| Namespace | Extension name |
|---|---|
| `Applications.Core` | `radius` |
| `Radius.Compute` | `radiusCompute` |
| `Radius.Data` | `radiusData` |
| `Radius.Security` | `radiusSecurity` |

Use `extension radiusCompute` — NOT `extension containerImages` or `extension containers`.

## app.bicep Structure (mandatory order)

Declare resources in this order (do NOT output this as code — it is only for your reference):

1. Extensions: `radius`, then `radiusCompute`, `radiusSecurity`, `radiusData` (only those needed)
2. Params: `environment`, then `@secure() password` if needed, then `@description(...) image` if needed
3. Application resource (`Applications.Core/applications@2023-10-01-preview`) — always exactly one
4. Data / infrastructure resources (databases, caches)
5. Secret resources (database credentials, API keys)
6. Container image resources (if building from Dockerfile)
7. Container resources (with connections to images and infra)
8. Routes (only if external ingress needed)

Rules:
- Always start with `extension radius` then namespace-level extensions in the fixed order, then params.
- Always declare exactly ONE `Applications.Core/applications@2023-10-01-preview` resource.
- If the app has a Dockerfile but no published image, add a `Radius.Compute/containerImages` resource. Use a `param image string` for the image reference (one image param per built image; name additional params `<serviceName>Image`). The container must reference the image via `<serviceName>Image.properties.image` and have a connection to `<serviceName>Image.id`.
- For database credentials, create a `Radius.Security/secrets` resource and reference it via `secretName` on the database resource.
- Use `@secure() param` for passwords — NEVER hardcode them.
- For each container service, emit exactly one `Radius.Compute/containers` resource.
- For each backing data store, emit the matching `Radius.Data/*` resource with an engine/instance-derived symbolic name.
- Only add `Radius.Compute/routes` if the app needs external ingress.

## Connections

Wire containers to infrastructure via `connections`. Read [connection-conventions.md](references/connection-conventions.md) for the correct env var format.

Rules:
- NEVER duplicate auto-injected env vars with manual `env` entries.
- Only add explicit `env` entries for app-specific variables NOT covered by connection auto-injection.

## Secrets

Read [secrets-handling.md](references/secrets-handling.md).

Each data store that needs credentials references its secret via `secretName: <engine>Secret.name` (e.g., `mysqlSecret.name`). The username is derived from the source database config (e.g., `MYSQL_USER`/`POSTGRES_USER` or a connection string); if none is found, use `<shortName>_user`. Use `@secure() param` for the password.

## Bicep Structure Rules

Read [bicep-structure-rules.md](references/bicep-structure-rules.md) for all structural rules.

## Validation Checklist

Before outputting, verify ALL:
- [ ] Application resource uses `Applications.Core/applications@2023-10-01-preview`
- [ ] Every `Radius.*` type matches a YAML schema in `resource-types-contrib`
- [ ] `Radius.*` types use `2025-08-01-preview`; `Applications.Core` uses `2023-10-01-preview`
- [ ] Extensions are in order: `radius`, `radiusCompute`, `radiusSecurity`, `radiusData`
- [ ] All names follow the Deterministic Naming Rules exactly
- [ ] `param environment string` is declared
- [ ] `@secure() param password string` declared if database credentials are needed
- [ ] `param image string` declared if building container images
- [ ] Exactly one `Applications.Core/applications` resource
- [ ] Each data store needing credentials references its secret via `secretName: <engine>Secret.name`
- [ ] Secret USERNAME is derived from source DB config (fallback `<shortName>_user`)
- [ ] Data store `database` name and `version` are derived from source, not hardcoded
- [ ] Symbolic names for data stores/secrets are engine-derived (no fixed `database`/`dbSecret`), unique per instance
- [ ] `connections` is a top-level property under `properties`, NOT inside `containers`
- [ ] `connections` is an object map, NOT an array
- [ ] Container images use `param image string`, not hardcoded
- [ ] Ports use `containerPort`, NOT `port`
- [ ] `build.context` is the Dockerfile's directory relative to repo root (`'.'` if at root)
- [ ] No comments or explanations in the generated Bicep
- [ ] No source analysis, step headings, or reasoning shown in chat

## Guardrails

- Use `Applications.Core/applications@2023-10-01-preview` — NOT `Radius.Core/applications`.
- Do NOT set readOnly properties.
- Do NOT reference readOnly properties of other resources in Bicep.
- Do NOT use array syntax where the schema specifies object maps.
- Do NOT place `connections` inside `containers`.
- Do NOT include comments in generated Bicep.
- Do NOT use a bare runtime base image when the app has a Dockerfile.
- Do NOT reuse a single symbolic name (like `database`/`dbSecret`) for multiple data stores — derive engine/role-based names (`mysqlDb`, `redisCache`, `mysqlSecret`) so multiple stores never collide.
- Do NOT hardcode a data store's `database` name or `version` — derive them from the source.
- Do NOT assume a single container is a "frontend" — name it from the app/service, not `-frontend`.
- Do NOT use `extension containerImages` or `extension containers` — use `extension radiusCompute`.
- Do NOT generate or output bicepconfig.json.
- ALWAYS create `Radius.Security/secrets` for database credentials.
- ALWAYS use `@secure() param` for passwords.
- ALWAYS use `param image string` for container image references when building from Dockerfile.

## Example

Read [todo-list-app-example.md](references/todo-list-app-example.md) for a complete worked example. The generated Bicep in that example is the **expected correct output** for `dockersamples/todo-list-app`.