# Architecture Patterns

Classify the application into exactly ONE of these patterns. This determines the resource composition.

## Pattern A: Stateless Web/API Service
- **Signals**: HTTP server, no database dependency, serves API or static content
- **Resources**: `Radius.Compute/containers` + optional `Radius.Compute/routes` for external ingress

## Pattern B: Stateful / Database-Backed Application
- **Signals**: HTTP server + database client library (mysql, mysql2, pg, sqlite3, mongoose, sequelize, prisma, redis, mssql, etc.)
- **Resources**: `Radius.Compute/containers` + `Radius.Data/*` (matching engine: `mySqlDatabases`, `postgreSqlDatabases`, `mongoDatabases`, `redisCaches`, `neo4jDatabases`, `sqlServerDatabases`) + `Radius.Security/secrets` (for DB credentials) + optional `Radius.Compute/routes`

## Pattern C: Event-Driven Application
- **Signals**: Message queue client (amqplib, kafkajs, sqs-consumer, bull, etc.), pub/sub patterns
- **Resources**: `Radius.Compute/containers` + messaging resource type (`Radius.Messaging/kafka` or `Radius.Messaging/rabbitMQ`)

## Pattern D: Batch Job
- **Signals**: No HTTP server, runs a task and exits, cron-like behavior
- **Resources**: `Radius.Compute/containers` with `restartPolicy: 'OnFailure'` or `'Never'`

## Pattern E: Streaming / Real-Time Processing Application
- **Signals**: WebSocket server (ws, socket.io), stream processing libraries
- **Resources**: `Radius.Compute/containers` + streaming resource type + optional `Radius.Compute/routes`

## How to classify

1. Check the package manifest for database client libraries → if present, **Pattern B**
2. Check for message queue libraries → if present, **Pattern C**
3. Check for HTTP server/framework → if present without DB, **Pattern A**
4. Check for streaming/WebSocket libraries → if present, **Pattern E**
5. If no HTTP server and runs to completion → **Pattern D**