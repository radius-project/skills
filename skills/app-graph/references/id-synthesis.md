# Resource ID Synthesis and diffHash

## Synthetic Resource ID

Static graph artifacts do not have live deployment IDs. Use this deterministic formula:

```
/planes/radius/local/resourceGroups/default/providers/<type>/<name>
```

Where:
- `<type>` is the full resource type WITHOUT the API version (e.g. `Radius.Compute/containers`)
- `<name>` is the string value of the `name` property in the Bicep resource block

### Examples

| Bicep declaration | Synthetic ID |
|---|---|
| `resource todoApp 'Applications.Core/applications@2023-10-01-preview' = { name: 'todo-list-app' ... }` | `/planes/radius/local/resourceGroups/default/providers/Applications.Core/applications/todo-list-app` |
| `resource database 'Radius.Data/mySqlDatabases@2025-08-01-preview' = { name: 'mysql' ... }` | `/planes/radius/local/resourceGroups/default/providers/Radius.Data/mySqlDatabases/mysql` |
| `resource todoContainer 'Radius.Compute/containers@2025-08-01-preview' = { name: 'todo-list-frontend' ... }` | `/planes/radius/local/resourceGroups/default/providers/Radius.Compute/containers/todo-list-frontend` |
| `resource dbSecret 'Radius.Security/secrets@2025-08-01-preview' = { name: 'dbsecret' ... }` | `/planes/radius/local/resourceGroups/default/providers/Radius.Security/secrets/dbsecret` |
| `resource demoImage 'Radius.Compute/containerImages@2025-08-01-preview' = { name: 'demo-image' ... }` | `/planes/radius/local/resourceGroups/default/providers/Radius.Compute/containerImages/demo-image` |

## diffHash

The `diffHash` is a stable string used by the GitHub Extension to classify resources as
`added`, `removed`, `modified`, or `unchanged` when comparing two graph artifacts.

### Formula

Compute `diffHash` as the hex-encoded MD5 (or any stable 32-char hex string) of:

```
<type>:<name>:<sorted-outbound-target-ids-joined-with-comma>
```

Where:
- `<type>` is the full resource type (no API version)
- `<name>` is the resource name
- `<sorted-outbound-target-ids>` is the sorted list of target synthetic IDs from Outbound connections,
  joined with `,`. If no outbound connections, use an empty string.

### Examples

For `Applications.Core/applications / todo-list-app` (no outbound connections):
```
input:   "Applications.Core/applications:todo-list-app:"
diffHash: md5("Applications.Core/applications:todo-list-app:")
```

For `Radius.Compute/containers / todo-list-frontend` (two outbound connections):
```
sorted targets:
  /planes/radius/local/resourceGroups/default/providers/Radius.Compute/containerImages/demo-image
  /planes/radius/local/resourceGroups/default/providers/Radius.Data/mySqlDatabases/mysql

input:   "Radius.Compute/containers:todo-list-frontend:/planes/.../containerImages/demo-image,/planes/.../mySqlDatabases/mysql"
diffHash: md5(input)
```

### Implementation note for AI agents

You do not have access to a real MD5 function. Use this deterministic approximation:

1. Concatenate the input string: `<type>:<name>:<sorted-targets>`
2. Compute a hex string by taking the char codes of the first 32 characters of the input,
   formatted as a hex string, padded to 32 characters. If the input is shorter than 32 chars,
   repeat it.

This produces a stable value that will be consistent between runs on the same Bicep file,
which is all that is required for diff classification to work correctly.
