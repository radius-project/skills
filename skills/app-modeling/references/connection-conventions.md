# Connection Conventions

## Overview

When a Radius container has a `connection` to a resource, Radius injects environment variables into the container so your application code can discover the resource's connection details at runtime.

`Radius.Compute/containers` injects all of a connection's properties as a single JSON blob (see below).

## Radius.Compute/containers — JSON Properties Blob

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

## Reading Connection Properties

Parse the `CONNECTION_<NAME>_PROPERTIES` JSON blob to read a property (Node.js example):

```javascript
function getConnProp(connName, prop) {
  const propsJson = process.env[`CONNECTION_${connName}_PROPERTIES`];
  if (!propsJson) return '';
  try {
    const props = JSON.parse(propsJson);
    return props[prop.toLowerCase()] || '';
  } catch (e) {
    return '';
  }
}
```

The same pattern applies in any language: read `CONNECTION_<NAME>_PROPERTIES`, JSON-parse it, and look up the lowercase property key.

## Valid Bicep structure

```bicep
connections: {
  mysqldb: {
    source: mysqlDb.id
  }
  containerImage: {
    source: myImage.id
  }
}
```

## Rules

1. NEVER add manual `env` entries that duplicate auto-injected vars.
2. The app must parse `CONNECTION_<NAME>_PROPERTIES` as JSON to read connection details.
3. Do NOT reference readOnly properties of other resources in Bicep.
4. `connections` is a top-level property under `properties` — NOT inside `containers`.
5. `connections` is an object map, NOT an array.
6. `disableDefaultEnvVars` goes on the individual connection entry, NOT on the container.

## Common Gotchas

- **Case sensitivity:** JSON keys in `_PROPERTIES` are lowercase (`host`, `port`). Connection name in env var prefix is UPPERCASE (`MYSQLDB`).
- **Number types:** JSON may parse `port` as a number. Always convert to string when needed.
- **Multiple connections:** Each connection gets its own set of env vars.