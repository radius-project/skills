# Bicep-to-Graph Parsing Rules

These rules define how to convert each Bicep `resource` declaration in `.radius/app.bicep`
into an `ApplicationGraphResource` for the static graph artifact.

## Step 1: Find all resource declarations

Scan the Bicep file for lines matching:

```
resource <symbolic> '<type>@<apiVersion>' = {
```

Each such declaration becomes one `ApplicationGraphResource`.

Record the **1-based line number** of this declaration for `appDefinitionLine`.

## Step 2: Extract `name`

Inside the resource block, find:

```bicep
name: '<value>'
```

The string value (without quotes) is the `name` field.

## Step 3: Extract `type`

From the resource declaration, take `<type>` (the part before `@`).

```
'Radius.Compute/containers@2025-08-01-preview'
 ^^^^^^^^^^^^^^^^^^^^^^^^^^^
 type = "Radius.Compute/containers"
```

Do NOT include the API version in `type`.

## Step 4: Set `provisioningState`

Always `"Succeeded"` for static graphs.

## Step 5: Parse connections

Look for a `connections: { ... }` block at the TOP LEVEL of `properties` (sibling of `containers`, NOT inside it).

Each key in the connections map is a connection name. The value has `source: <symbolic>.id`.

```bicep
connections: {
  mysqldb: {
    source: database.id      // ← target symbolic name is "database"
  }
  demoContainerImage: {
    source: demoImage.id     // ← target symbolic name is "demoImage"
  }
}
```

For each entry:
1. Resolve `<symbolic>` to the corresponding resource's `name` and `type`
   by looking up the symbolic name in your parsed resource list.
2. Compute the target resource's synthetic ID using [id-synthesis.md](id-synthesis.md).
3. Emit an **Outbound** connection entry on the declaring resource:
   ```json
   { "id": "<target-synthetic-id>", "direction": "Outbound" }
   ```
4. Emit an **Inbound** connection entry on the target resource:
   ```json
   { "id": "<declaring-resource-synthetic-id>", "direction": "Inbound" }
   ```

## Step 6: Set `outputResources`

Always `[]` for static graphs.

## Step 7: Compute `diffHash` and `appDefinitionLine`

See [id-synthesis.md](id-synthesis.md).

## Parsing edge cases

### Connections inside `containers` map (wrong location)
If the Bicep was authored incorrectly with `connections` inside the `containers` block,
still parse it and emit the correct graph entries. Connections are always between
top-level resources regardless of where they appear in the Bicep.

### `param` references in connections
Ignore any `source` that references a param (e.g. `source: environment`).
Only emit connection entries for `source: <symbolic>.id` where `<symbolic>` is a `resource`.

### `containerImages` connection
The `demoContainerImage` connection to a `Radius.Compute/containerImages` resource
is a valid graph edge. Emit it like any other connection.

### `Applications.Core/applications`
This resource has no `connections` block. It always has `connections: []` in the graph.

### `Radius.Security/secrets`
Secrets are referenced indirectly via `secretName: dbSecret.name` on database resources,
NOT via `connections`. Do NOT emit a connection for this reference.
Only `connections: { source: <symbolic>.id }` entries generate graph edges.

## Full parsing example

Given this Bicep (simplified):

```bicep
extension radius                              // line 1
extension radiusCompute                       // line 2
extension radiusSecurity                      // line 3
extension radiusData                          // line 4
                                              // line 5
param environment string                      // line 6
                                              // line 7
resource todoApp 'Applications.Core/applications@2023-10-01-preview' = {  // line 8
  name: 'todo-list-app'                       // line 9
  properties: {                               // line 10
    environment: environment                  // line 11
  }                                           // line 12
}                                             // line 13
                                              // line 14
resource database 'Radius.Data/mySqlDatabases@2025-08-01-preview' = {  // line 15
  name: 'mysql'                               // line 16
  ...                                         
}                                             // line 23
                                              // line 24
resource dbSecret 'Radius.Security/secrets@2025-08-01-preview' = {  // line 25
  name: 'dbsecret'                            
  ...
}                                             // line 38
                                              
resource demoImage 'Radius.Compute/containerImages@2025-08-01-preview' = {  // line 40
  name: 'demo-image'                          
  ...
}                                             // line 49
                                              
resource todoContainer 'Radius.Compute/containers@2025-08-01-preview' = {  // line 51
  name: 'todo-list-frontend'
  properties: {
    ...
    connections: {
      mysqldb: {
        source: database.id
      }
      demoContainerImage: {
        source: demoImage.id
      }
    }
  }
}
```

Produces these resources:

| symbolic | name | type | appDefinitionLine | outbound connections |
|---|---|---|---|---|
| `todoApp` | `todo-list-app` | `Applications.Core/applications` | 8 | none |
| `database` | `mysql` | `Radius.Data/mySqlDatabases` | 15 | none (receives Inbound from todoContainer) |
| `dbSecret` | `dbsecret` | `Radius.Security/secrets` | 25 | none |
| `demoImage` | `demo-image` | `Radius.Compute/containerImages` | 40 | none (receives Inbound from todoContainer) |
| `todoContainer` | `todo-list-frontend` | `Radius.Compute/containers` | 51 | → mysql, → demo-image |
