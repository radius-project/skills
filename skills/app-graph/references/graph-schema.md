# Application Graph Artifact Schema

The `StaticGraphArtifact` is the JSON file stored on the `radius-graph` orphan branch.
It is the complete self-contained representation of the application graph derived from
a Bicep file, consumed by the Radius GitHub Extension for rendering.

## StaticGraphArtifact

```json
{
  "version": "1.0.0",
  "generatedAt": "<ISO 8601 timestamp>",
  "sourceFile": ".radius/app.bicep",
  "application": {
    "resources": [ /* ApplicationGraphResource[] */ ]
  }
}
```

| Field | Type | Description |
|---|---|---|
| `version` | `string` | Always `"1.0.0"` |
| `generatedAt` | `string` | ISO 8601 UTC timestamp of generation (e.g. `"2025-05-14T10:00:00.000Z"`) |
| `sourceFile` | `string` | Repo-root-relative path to the Bicep source: `".radius/app.bicep"` |
| `application` | `ApplicationGraphResponse` | The graph data |

## ApplicationGraphResponse

```json
{
  "resources": [ /* ApplicationGraphResource[] */ ]
}
```

## ApplicationGraphResource

```json
{
  "id": "/planes/radius/local/resourceGroups/default/providers/Applications.Core/applications/todo-list-app",
  "name": "todo-list-app",
  "type": "Applications.Core/applications",
  "provisioningState": "Succeeded",
  "connections": [
    { "id": "<target-resource-id>", "direction": "Outbound" }
  ],
  "outputResources": [],
  "appDefinitionLine": 12,
  "diffHash": "abc123..."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | `string` | ✅ | Synthetic resource ID — see [id-synthesis.md](id-synthesis.md) |
| `name` | `string` | ✅ | Value of the `name` property in the Bicep resource block |
| `type` | `string` | ✅ | Full resource type **without** API version (e.g. `"Radius.Compute/containers"`) |
| `provisioningState` | `string` | ✅ | Always `"Succeeded"` for static graphs |
| `connections` | `ApplicationGraphConnection[]` | ✅ | Inbound and outbound connections to/from other resources |
| `outputResources` | `ApplicationGraphOutputResource[]` | ✅ | Always `[]` for static graphs (no live deployment) |
| `appDefinitionLine` | `number` | ✅ | 1-based line number of the `resource` declaration in `app.bicep` |
| `diffHash` | `string` | ✅ | Stable hash for diff classification — see [id-synthesis.md](id-synthesis.md) |
| `codeReference` | `string` | ❌ | Optional path to application source code (not set for static graphs) |

## ApplicationGraphConnection

```json
{ "id": "<resource-id>", "direction": "Outbound" }
```

| Field | Type | Values |
|---|---|---|
| `id` | `string` | Synthetic ID of the connected resource |
| `direction` | `"Inbound"` \| `"Outbound"` | `"Outbound"` on the resource that declares the connection; `"Inbound"` on the target |

## ApplicationGraphOutputResource

For static graphs this array is always empty. In live deployments it would contain Kubernetes
or cloud resources (Deployments, Services, etc.) provisioned by the recipe.

## Complete example

```json
{
  "version": "1.0.0",
  "generatedAt": "2025-05-14T10:00:00.000Z",
  "sourceFile": ".radius/app.bicep",
  "application": {
    "resources": [
      {
        "id": "/planes/radius/local/resourceGroups/default/providers/Applications.Core/applications/todo-list-app",
        "name": "todo-list-app",
        "type": "Applications.Core/applications",
        "provisioningState": "Succeeded",
        "connections": [],
        "outputResources": [],
        "appDefinitionLine": 9,
        "diffHash": "e3b0c44298fc1c149afbf4c8996fb924"
      },
      {
        "id": "/planes/radius/local/resourceGroups/default/providers/Radius.Data/mySqlDatabases/mysql",
        "name": "mysql",
        "type": "Radius.Data/mySqlDatabases",
        "provisioningState": "Succeeded",
        "connections": [
          {
            "id": "/planes/radius/local/resourceGroups/default/providers/Radius.Compute/containers/todo-list-frontend",
            "direction": "Inbound"
          }
        ],
        "outputResources": [],
        "appDefinitionLine": 20,
        "diffHash": "a1b2c3d4e5f6..."
      },
      {
        "id": "/planes/radius/local/resourceGroups/default/providers/Radius.Compute/containers/todo-list-frontend",
        "name": "todo-list-frontend",
        "type": "Radius.Compute/containers",
        "provisioningState": "Succeeded",
        "connections": [
          {
            "id": "/planes/radius/local/resourceGroups/default/providers/Radius.Data/mySqlDatabases/mysql",
            "direction": "Outbound"
          },
          {
            "id": "/planes/radius/local/resourceGroups/default/providers/Radius.Compute/containerImages/demo-image",
            "direction": "Outbound"
          }
        ],
        "outputResources": [],
        "appDefinitionLine": 55,
        "diffHash": "b2c3d4e5f6a1..."
      }
    ]
  }
}
```
