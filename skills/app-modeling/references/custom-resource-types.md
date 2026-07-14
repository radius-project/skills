# Custom Resource Type Generation

Use this workflow when the selected deployment profile requires a backing service that has no compatible standard type in `radius-project/resource-types-contrib`. It generates a portable developer contract with an Azure-only implementation and stores every artifact in the application repository's `.radius/` directory.

## Required artifact set

Generate this exact layout:

```text
.radius/
├── app.bicep
├── bicepconfig.json
├── custom-recipe-pack.bicep
├── custom-types.tgz
├── custom-types.yaml
└── recipes/
    └── <type-name>.bicep
```

Use one Recipe file per custom type. Keep all custom type schemas in the single `custom-types.yaml` manifest and all published Recipe references in the single `custom-recipe-pack.bicep`.

Do not write custom artifacts under `~/.radius/`, outside the application repository, or on a different branch.

## Eligibility and evidence

Create a custom type only when all of these are true:

1. Source evidence proves the component, protocol or SDK contract, native configuration keys, required connection values, and selected deployment profile.
2. No type in the standard catalog models that contract.
3. An Azure service implements the contract without requiring an unrequested application rewrite. A similar category is not enough; for example, do not replace NATS with Service Bus or Memcached with Redis.
4. The exact Azure Bicep resource or pinned module contract can be verified, including required inputs, output paths, API version, networking, TLS, and authentication.
5. Every app-facing output can be represented as a nonsecret read-only property or a managed-secret key.
6. The target Environment is intended to use an Azure provider. Ask the user when provider selection is ambiguous.

If any condition fails, report the component as unsupported. Do not generate partial custom artifacts or a Recipe Pack entry that cannot deploy.

## Repository and publication identity

Before generating files:

1. Confirm the working directory is a Git repository with a GitHub remote.
2. Read `<owner>/<repo>` from the GitHub remote; lowercase both segments for GHCR.
3. Read the current branch with `git branch --show-current`.
4. Confirm it is the existing Copilot worktree branch and is not empty, detached, or the repository's default branch.
5. Do not create, rename, or switch branches. All generated files must be committed and pushed to this branch.
6. Confirm `rad`, `git`, `gh`, and a GHCR-compatible credential helper such as Docker are available before making remote changes. Use `bicep` when it is on `PATH`; otherwise run `rad bicep download` and use the Radius-managed compiler at `$HOME/.rad/bin/bicep`.

If the repository remote or current branch cannot be established unambiguously, ask the user instead of guessing.

## Custom type manifest

The path is `.radius/custom-types.yaml`.

Use the fixed namespace `Radius.Resources` so one generated extension can contain every missing type. Type names are plural camelCase and application-oriented, such as `mqttBrokers` or `vectorDatabases`; never include `Azure`, an ARM provider namespace, or a SKU in the type name.

Use API version `2025-08-01-preview`. Merge additional types into an existing compatible manifest without deleting or rewriting unrelated types. If an existing type name has an incompatible schema, ask the user before changing it.

Each schema must:

- define `environment` as a required string;
- define `application` as an optional string unless the resource can never be shared;
- expose only portable developer intent derived from source;
- include a top-level developer-facing description with a Bicep usage example and connection guidance;
- include descriptions for every property, identifying required, optional, and read-only fields;
- mark Recipe-populated values `readOnly: true`;
- mark developer-supplied secret strings `x-radius-sensitive: true`; and
- model Recipe-generated secrets with the standard read-only `secrets` object containing a reserved `name` plus the declared secret keys.

Use this shape and remove properties the application does not need:

```yaml
namespace: Radius.Resources
types:
  <typeName>:
    description: |
      The Radius.Resources/<typeName> Resource Type ...
    apiVersions:
      '2025-08-01-preview':
        schema:
          type: object
          properties:
            environment:
              type: string
              description: (Required) The Radius Environment ID.
            application:
              type: string
              description: (Optional) The Radius Application ID.
            <developerProperty>:
              type: string
              description: (Required) Portable application configuration.
            <connectionProperty>:
              type: string
              description: (Read Only) Connection value populated by the Recipe.
              readOnly: true
            secrets:
              type: object
              description: (Read Only) Recipe-generated secrets exposed through a managed Radius secret.
              properties:
                name:
                  type: string
                  description: (Read Only) Name of the managed Radius secret.
                  readOnly: true
                <secretKey>:
                  type: string
                  description: (Read Only) Sensitive connection value populated by the Recipe.
                  readOnly: true
          required:
            - environment
            - <developerProperty>
```

Omit `secrets` when the Recipe has no secret outputs. Never expose a generated credential as an ordinary read-only property.

## Bicep extension

After the manifest is final, generate the local extension:

```bash
rad bicep publish-extension \
  --from-file .radius/custom-types.yaml \
  --target .radius/custom-types.tgz \
  --force
```

This command must succeed. Never hand-author, copy, or fake `custom-types.tgz`, and regenerate it after every schema change.

## Bicep configuration

The path is `.radius/bicepconfig.json`, next to `app.bicep`.

- If it already exists, preserve every key and merge the extension entry.
- If a configuration exists in an ancestor directory, copy its complete JSON object into `.radius/bicepconfig.json`, rebase any relative local paths to the new directory, then merge the entry.
- If no configuration exists, create one with the `radius` extension matching the installed Radius release. Use `br:biceptypes.azurecr.io/radius:latest` only for an edge installation where `latest` is the matching release contract.
- Add exactly `"customTypes": "custom-types.tgz"` under `extensions`.
- If `customTypes` already points elsewhere, stop and ask before replacing it.
- Keep valid JSON and do not remove module aliases, analyzers, cloud settings, or experimental feature settings.

When custom types are used, `.radius/app.bicep` begins with:

```bicep
extension radius
extension customTypes
```

## Azure Recipe

Create `.radius/recipes/<type-name>.bicep` for each custom type. A Recipe is Radius-aware Azure Bicep, not a direct Recipe Pack mapping to a generic module.

Every Recipe must:

- declare `param context object`;
- deploy only the Azure resources needed for the custom type;
- use verified, supported Azure API versions or explicitly pinned Bicep modules;
- derive stable, repeatable, Azure-valid names from `context.resource.name`, `context.resource.id`, and `uniqueString`; never use `utcNow()` for identity;
- use the Environment's configured Azure resource-group scope and `resourceGroup().location` unless the selected service requires a proven alternative;
- apply Radius ownership tags when the Azure resource supports tags;
- map developer inputs only from exact `context.resource.properties.<name>` paths defined by the schema;
- avoid public networking, local authentication, broad firewall rules, or weak TLS unless the selected profile explicitly requires and justifies them;
- return every nonsecret read-only schema property under `result.values`;
- return every generated secret under `result.secrets` with an exact schema key; and
- include no literal credentials or tokens.

Use the standard result contract:

```bicep
@description('Radius-provided context for the Recipe.')
param context object

@description('Azure region for provisioned resources.')
param location string = resourceGroup().location

var radiusTags = {
  'radapp.io-environment': context.environment.id
  'radapp.io-application': context.application == null ? '' : context.application.id
  'radapp.io-resource': context.resource.id
}

// Verified Azure resource or pinned module declarations.

output result object = {
  values: {
    <connectionProperty>: <verifiedOutput>
  }
  secrets: {
    #disable-next-line outputs-should-not-contain-secrets
    <secretKey>: <verifiedSecretOutput>
  }
}
```

Omit the `secrets` map when empty. Azure resource IDs are tracked automatically for Azure Bicep Recipes; include a `resources` list only when the Recipe creates resources Radius cannot infer.

Compile each Recipe without leaving build output:

```bash
<bicep> build .radius/recipes/<type-name>.bicep --stdout >/dev/null
```

Compilation warnings about unknown resources, missing properties, or secret handling are failures.

## GHCR publication

Publish each validated Recipe to the current repository's GHCR namespace only after the user approves the exact targets together with the planned commit and push.

1. Compute the lowercase SHA-256 digest of the final Recipe file and use the first 12 hexadecimal characters as `<digest>`.
2. Lowercase the custom type name for the OCI package path (`mqttBrokers` becomes `<type-package>` value `mqttbrokers`). Keep the original camelCase name in the schema, Bicep resource type, Recipe filename, and Recipe Pack key.
3. Set the target to:

   ```text
   br:ghcr.io/<owner>/<repo>/radius-recipes/<type-package>:sha256-<digest>
   ```

4. Authenticate to `ghcr.io` without printing or storing the token. The credential must have package write permission for `<owner>`.
5. Publish with:

   ```bash
   rad bicep publish \
     --file .radius/recipes/<type-name>.bicep \
     --target br:ghcr.io/<owner>/<repo>/radius-recipes/<type-package>:sha256-<digest>
   ```

6. Record the Recipe Pack source without the `br:` prefix:

   ```text
   ghcr.io/<owner>/<repo>/radius-recipes/<type-package>:sha256-<digest>
   ```

Never use `latest`, a branch name, an unpinned tag, or a registry outside the current GitHub repository's owner/repo path. Do not continue to commit or PR creation if publication fails. Do not change package visibility automatically; report when the target Environment needs registry credentials.

## Custom Recipe Pack

The path is `.radius/custom-recipe-pack.bicep`.

Generate one Recipe Pack containing one entry per custom type:

```bicep
extension radius

resource customRecipePack 'Radius.Core/recipePacks@2025-08-01-preview' = {
  name: 'custom-recipe-pack'
  location: 'global'
  properties: {
    recipes: {
      'Radius.Resources/<type-name>': {
        kind: 'bicep'
        source: 'ghcr.io/<owner>/<repo>/radius-recipes/<type-package>:sha256-<digest>'
      }
    }
  }
}
```

Merge into an existing generated pack without deleting unrelated custom entries. One fully qualified type key may appear only once. A conflicting source requires user confirmation.

Do not create or modify an Environment resource in this file. A platform operator deploys this pack and attaches it alongside the Environment's existing Recipe Packs.

Do not register `custom-types.yaml`, deploy the Recipe Pack, or mutate a live Radius Environment automatically. Those are platform operations separate from generating and publishing the repository artifacts.

## Application wiring

Declare each generated type with the exact namespace, name, API version, and properties from `custom-types.yaml`. Set only developer-authored properties. Bind Recipe outputs using the same runtime-contract rules as standard types:

- nonsecret values may use exact read-only property references or a verified generic connection contract;
- generated secrets use the managed secret name and exact key declared by the custom schema; and
- every value must reach the application's exact native configuration key and format.

## Validation and branch publication

Before committing:

1. Confirm `custom-types.yaml` parses and contains exactly the intended contracts.
2. Regenerate `custom-types.tgz` from the final manifest.
3. Compile every Recipe and `custom-recipe-pack.bicep` with `<bicep> build <file> --stdout`.
4. Publish every Recipe and confirm each Recipe Pack source matches its successful target.
5. Compile `.radius/app.bicep` with `<bicep> build .radius/app.bicep --stdout`; Bicep must discover `.radius/bicepconfig.json`.
6. Confirm the application, schema, Recipe result, Recipe Pack, and runtime contract agree on every property and secret key.
7. Inspect `git status` and ensure no credentials, build outputs, or unrelated files are included.

Stage the intended `.radius/` files explicitly, including `custom-types.tgz` if it is ignored. Show the user the exact GHCR publication targets, staged files, proposed commit message, branch, and remote, and obtain explicit approval. Then publish the Recipes, rerun the final checks, commit the staged files on the existing Copilot worktree branch, and push that same branch to its GitHub remote. Never create, rename, or switch branches, and never force-push over unrelated remote work.

Only after the push succeeds should the response offer to open a pull request against `main`.
