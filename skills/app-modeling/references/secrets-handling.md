# Secrets Handling

## Rules

- NEVER hardcode passwords, tokens, or keys in `app.bicep`.
- For secrets passed at deploy time, use `@secure() param` and pass via `rad deploy -p`.
- ALWAYS create a `Radius.Security/secrets` resource for database credentials (username, password).
- Create one secret per data store that needs credentials; symbolic `<engine>Secret`, name `'<engine>-secret'` (e.g., `mysqlSecret` / `'mysql-secret'`).
- Reference the secret from the database resource via `secretName: mySecret.name`.
- Derive the database USERNAME from the source config (`MYSQL_USER`, `POSTGRES_USER`, connection string); if absent, use `<shortName>_user`. Never invent a username unrelated to the source.
- Use `Radius.Security/secrets` for any app-specific secrets (API keys, TLS certs) as well.

## Database credentials pattern

```bicep
@secure()
param password string

resource mysqlSecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'mysql-secret'
  properties: {
    environment: environment
    application: app.id
    data: {
      USERNAME: {
        value: 'todo_user'
      }
      PASSWORD: {
        value: password
      }
    }
  }
}

resource mysqlDb 'Radius.Data/mySqlDatabases@2025-08-01-preview' = {
  name: 'mysql'
  properties: {
    environment: environment
    application: app.id
    database: 'todos'      // derived from source (e.g. MYSQL_DATABASE)
    version: '8.0'         // derived from source (e.g. image tag mysql:8.0)
    secretName: mysqlSecret.name
  }
}
```

Deploy: `rad deploy app.bicep -p password=$DB_PASSWORD -p image=ghcr.io/myorg/myapp:latest`

## App-specific secrets pattern

```bicep
@secure()
param apiKey string

resource appSecrets 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'app-secrets'
  properties: {
    environment: environment
    application: app.id
    data: {
      API_KEY: {
        value: apiKey
      }
    }
  }
}
```

## Common mistakes to avoid

- Do NOT write `password: 'mysecretpassword'` — always use `@secure() param`
- Do NOT skip creating `Radius.Security/secrets` for database credentials
- Do NOT forget to add `secretName: <engine>Secret.name` (e.g., `mysqlSecret.name`) on the database resource
- Keys in `data` are UPPERCASE (`USERNAME`, `PASSWORD`, `API_KEY`)