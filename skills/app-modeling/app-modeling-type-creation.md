# Creating a Radius resource type

Reference for `app-modeling.md`. Use this when modeling an application requires a dependency
that has **no** existing Radius resource type — neither a default type nor any other registered
Radius resource type.

Authoring a type should be the exception, not the default. Always prefer an existing type.
Create a new one only when a real dependency cannot otherwise be modeled, and the user wants
to proceed rather than drop the dependency.

## When to create a type

1. Confirm no default type fits (see `app-modeling-radius-usage.md`).
2. Confirm no other registered Radius resource type fits (check the catalog in
   `app-modeling-available-resource-types.md`).
3. Only then author a new type. If a type is not warranted, return the unsupported-dependency
   response (R3) from `app-modeling-compatible-applications.md`.

## What a resource type is

A resource type is an application-oriented **schema** — the contract a developer uses in
`app.bicep`. It defines the properties developers set (inputs) and the properties a Recipe
fills in after provisioning (read-only outputs). A type does not itself provision anything;
a **Recipe** (Bicep or Terraform, registered on the environment) implements it. A type with
no recipe will not deploy.

## Manifest schema

Define the type in a YAML manifest. Keep it simple and application-focused; avoid
infrastructure- or platform-specific knobs.

```yaml
namespace: Radius.<Category>      # e.g. Radius.Data, Radius.Compute, Radius.Messaging
types:
  <typeName>:                     # camelCase, plural — e.g. redisCaches, postgreSqlDatabases
    description: |
      Developer-facing Markdown docs, surfaced to developers in the Radius Dashboard.
      Include a short usage example in a fenced code block.
    apiVersions:
      '2025-08-01-preview':       # date the type is authored/validated; format YYYY-MM-DD-preview
        schema:
          type: object
          properties:
            environment:
              type: string
              description: (Required) The Radius Environment ID. Typically value should be `environment`.
            application:
              type: string
              description: (Optional) The Radius Application ID.
            capacity:
              type: string
              enum: [S, M, L]
              description: (Optional) The capacity of the resource. The value is assumed to be `S` if not set.
            host:
              type: string
              readOnly: true
              description: (Read Only) The host used to connect to the resource.
            port:
              type: string
              readOnly: true
              description: (Read Only) The port used to connect to the resource.
            secrets:
              type: object
              properties:
                password:
                  type: string
                  readOnly: true
                  x-radius-sensitive: true
                  description: (Read Only) The password used to connect to the resource.
          required:
            - environment
```

## Schema guidelines

| Rule | Detail |
|---|---|
| `namespace` | `Radius.<Category>` — e.g. `Radius.Data`, `Radius.Compute`, `Radius.Security` |
| Type name | camelCase **and** plural — `redisCaches`, `sqlDatabases`, `rabbitMQQueues` |
| API version | `YYYY-MM-DD-preview`; drop `-preview` only at Stable maturity |
| `environment` | Always present and always in `required` |
| `application` | In `required` only if the resource must always belong to an app; omit for shared resources |
| Property names | camelCase, each with a `description` |
| `readOnly: true` | For properties set only by the Recipe after deploy (host, port, username) |
| `x-radius-sensitive: true` | For passwords, tokens, keys — Radius encrypts these |
| Valid `type` values | `string`, `integer`, `object`, `array` (use `enum:` on a `string` for enumerations) |
| Descriptions | Prefix with `(Required)`, `(Optional)`, or `(Read Only)`; wrap example values in backticks |

Keep types developer-oriented. Do not leak platform-specific or infrastructure-specific
properties into the schema.

## Make the type available

The manifest is the artifact you produce. Radius registers a resource type from its manifest;
registration and deployment happen in the Radius control plane, outside this environment. Place
the manifest with the project, or contribute it upstream (see Contributing back). Once the type
is registered, reference it in `.radius/app.bicep` using the exact namespace, type name, and
`apiVersion` from the manifest, then validate per `app-modeling-validation.md`.

## Recipes (to make it deployable)

A registered type still needs a Recipe to provision real infrastructure. A Recipe is a Bicep
or Terraform template that receives a `context` object from Radius and outputs `result` with
`resources` (for lifecycle tracking) and `values` (which populate the type's `readOnly`
properties). Recipes are registered on the environment as a **recipe pack**, not in `app.bicep`.
See `app-modeling-recipe-pack.md` for the recipe and recipe-pack format, plus a default Azure
recipe pack covering the supported types.
