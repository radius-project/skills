# Connection Conventions

## Overview

When a Radius container has a `connection` to a resource, Radius injects environment variables into the container so your application code can discover the resource's connection details at runtime.

**The format of these environment variables differs** depending on whether you use `Applications.Core/containers` (built-in) or `Radius.Compute/containers` (recipe-based).

## Radius.Compute/containers (recipe-based) — JSON Properties Blob

All properties are packed into a single JSON environment variable:

```
CONNECTION_<NAME>_PROPERTIES={"host":"...","port":"...","database":"...","username":"...","password":"..."}
CONNECTION_<NAME>_ID=<resource-id>
CONNECTION_<NAME>_NAME=<connection-name>
CONNECTION_<NAME>_TYPE=<resource-type>
```

> Property names in the JSON blob are **lowercase** (matching the resource type schema). The connection name in the env var prefix is **UPPERCASE**.

### Example

Connection named `mysqldb` to `Radius.Data/mySqlDatabases`:

| Variable | Example Value |
|----------|---------------|
| `CONNECTION_MYSQLDB_PROPERTIES` | `{"database":"todos","username":"todo_user","password":"abc123","version":"8.0","host":"mysql-svc.default.svc.cluster.local","port":"3306"}` |
| `CONNECTION_MYSQLDB_ID` | `/planes/radius/local/.../Radius.Data/mySqlDatabases/mysql` |
| `CONNECTION_MYSQLDB_NAME` | `mysqldb` |
| `CONNECTION_MYSQLDB_TYPE` | `Radius.Data/mySqlDatabases` |

## Applications.Core/containers (built-in) — Individual Env Vars

Each readOnly property becomes a separate environment variable:

```
CONNECTION_<NAME>_HOST
CONNECTION_<NAME>_PORT
CONNECTION_<NAME>_DATABASE
CONNECTION_<NAME>_USERNAME
CONNECTION_<NAME>_PASSWORD
```

> This format is NOT used by `Radius.Compute/containers`.

## Writing Portable Application Code

To support both connection formats, check for `_PROPERTIES` first, then fall back to individual vars:

### Node.js

```javascript
function getConnProp(connName, prop) {
  const propsJson = process.env[`CONNECTION_${connName}_PROPERTIES`];
  if (propsJson) {
    try {
      const props = JSON.parse(propsJson);
      return props[prop.toLowerCase()] || '';
    } catch (e) { /* fall through */ }
  }
  return process.env[`CONNECTION_${connName}_${prop}`] || '';
}
```

### Go

```go
func getConnProp(connName, prop string) string {
    propsJSON := os.Getenv("CONNECTION_" + connName + "_PROPERTIES")
    if propsJSON != "" {
        var props map[string]interface{}
        if err := json.Unmarshal([]byte(propsJSON), &props); err == nil {
            if val, ok := props[strings.ToLower(prop)]; ok {
                return fmt.Sprintf("%v", val)
            }
        }
    }
    return os.Getenv("CONNECTION_" + connName + "_" + prop)
}
```

### Python

```python
import json, os

def get_conn_prop(conn_name: str, prop: str) -> str:
    props_json = os.getenv(f"CONNECTION_{conn_name}_PROPERTIES", "")
    if props_json:
        try:
            props = json.loads(props_json)
            return str(props.get(prop.lower(), ""))
        except json.JSONDecodeError:
            pass
    return os.getenv(f"CONNECTION_{conn_name}_{prop}", "")
```

## Valid Bicep structure

```bicep
connections: {
  mysqldb: {
    source: database.id
  }
  containerImage: {
    source: myImage.id
  }
}
```

## Rules

1. NEVER add manual `env` entries that duplicate auto-injected vars.
2. When using `Radius.Compute/containers`, the app must parse `CONNECTION_<NAME>_PROPERTIES` as JSON.
3. Do NOT reference readOnly properties of other resources in Bicep.
4. `connections` is a top-level property under `properties` — NOT inside `containers`.
5. `connections` is an object map, NOT an array.
6. `disableDefaultEnvVars` goes on the individual connection entry, NOT on the container.

## Common Gotchas

- **Case sensitivity:** JSON keys in `_PROPERTIES` are lowercase (`host`, `port`). Connection name in env var prefix is UPPERCASE (`MYSQLDB`).
- **Number types:** JSON may parse `port` as a number. Always convert to string when needed.
- **Multiple connections:** Each connection gets its own set of env vars.