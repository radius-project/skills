# Example: Todo-List-App (dockersamples/todo-list-app)

## Source analysis

- **Role**: long-running Node.js/Express web service
- **Listener**: port 3000, configured by the application
- **Persistence**: SQLite by default; MySQL when `MYSQL_HOST` is present
- **Native configuration read by source**: `MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DB`
- **Backing service**: MySQL 8.0 with database `todos`
- **Image**: complete Dockerfile/build context, pinned to an immutable source commit
- **Storage**: the modeled MySQL service owns persistence; no application filesystem volume is required
- **Primary pattern**: Web App

## Key decisions

1. The unmodified source does not parse Radius generic `CONNECTION_*` variables, so the container maps every required native MySQL variable explicitly.
2. `mysqlDb.properties.host` is a nonsecret read-only output confirmed in the exact MySQL schema. Referencing it creates the database dependency edge.
3. The developer-supplied password enters through `@secure()` and the MySQL schema's sensitive `password` property. A `Radius.Security/secrets` resource also exposes it to the workload through `secretKeyRef`; it is not assigned to plain `env.value`.
4. A generic connection is omitted because the source does not consume it and direct references already establish ordering.
5. The source build is consumed through `todoImage.properties.imageReference`.
6. `containerPort: 3000` matches the inspected process listener. No route is added because external ingress was not requested.

## Expected app.bicep

```bicep
extension radius

param environment string

@secure()
param mysqlPassword string

var databaseName = 'todos'
var databaseUsername = 'myadmin'

resource todoApp 'Radius.Core/applications@2025-08-01-preview' = {
  name: 'todo-list-app'
  properties: {
    environment: environment
  }
}

resource mysqlDb 'Radius.Data/mySqlDatabases@2025-08-01-preview' = {
  name: 'mysql'
  properties: {
    environment: environment
    application: todoApp.id
    database: databaseName
    version: '8.0'
    username: databaseUsername
    password: mysqlPassword
  }
}

resource mysqlRuntimeSecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'mysql-runtime-secret'
  properties: {
    environment: environment
    application: todoApp.id
    data: {
      MYSQL_PASSWORD: {
        value: mysqlPassword
      }
    }
  }
}

resource todoImage 'Radius.Compute/containerImages@2025-08-01-preview' = {
  name: 'todo-list-app-image'
  properties: {
    environment: environment
    application: todoApp.id
    build: {
      source: 'git::https://github.com/dockersamples/todo-list-app.git?ref=5a6fbf5caf982f1d928fe6c1c32aa74f1e95e063'
    }
  }
}

resource todoContainer 'Radius.Compute/containers@2025-08-01-preview' = {
  name: 'todo-list-app'
  properties: {
    environment: environment
    application: todoApp.id
    containers: {
      todo: {
        image: todoImage.properties.imageReference
        ports: {
          web: {
            containerPort: 3000
          }
        }
        env: {
          MYSQL_HOST: {
            value: mysqlDb.properties.host
          }
          MYSQL_USER: {
            value: databaseUsername
          }
          MYSQL_PASSWORD: {
            valueFrom: {
              secretKeyRef: {
                secretName: mysqlRuntimeSecret.name
                key: 'MYSQL_PASSWORD'
              }
            }
          }
          MYSQL_DB: {
            value: databaseName
          }
        }
      }
    }
  }
}
```
