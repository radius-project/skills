# Runtime Contract

A compiling `app.bicep` proves only that its syntax and resource shapes are accepted. The generated model must also satisfy every workload's real process, configuration, storage, and dependency contract.

## Evidence to inspect

Inspect primary evidence before relying on prose documentation:

1. Dockerfile and complete build context: stages, copied files, build arguments, `ENTRYPOINT`, `CMD`, `WORKDIR`, `USER`, installed shells/tools, and writable paths.
2. Compose/Kubernetes manifests: service roles, images, commands, environment, ports, volumes, health checks, and dependency topology.
3. Entrypoints and source: environment/config reads, defaults, client constructors, URL assembly, protocol options, listener address/port, migrations, and worker-versus-web behavior.
4. Example configuration and documentation: use these to confirm source behavior, not to override it.

`EXPOSE`, a compose port mapping, or a health endpoint alone is not proof that the process listens correctly or can reach its dependencies.

## Inventory each workload

Record this contract for every executable role:

| Field | Questions to answer |
|---|---|
| Role | Long-running web service, worker, scheduler, migration/init job, sidecar, or one-shot CLI? |
| Image | Complete source build or pinned published image? Which immutable commit, tag, or digest? |
| Process | What do the image entrypoint and command run? Is an override required and does the image contain the required executable or shell? |
| Listener | Which address, port, and protocol does the process actually use? Which setting configures it? |
| Configuration | Which exact environment variables, flags, files, nesting syntax, casing, and defaults are consumed? |
| Dependencies | Which hosts, ports, database/topic/queue names, credentials, URLs, TLS modes, and protocol versions does the client require? |
| Secrets | Which values are supplied by the developer and which are produced by a recipe? Can the container consume them by reference? |
| Storage | Which paths must be writable or persistent? What ownership and access mode does the process require? |

Model separate web, worker, producer, consumer, and init roles separately even when they share an image.

## Map every required value

For each required app-native input, choose exactly one supported source:

- explicit nonsecret `env.value` from a literal or a verified resource output;
- `valueFrom.secretKeyRef` using an exact secret resource and key;
- runtime composition from previously bound values when the app requires a larger URL/config value; or
- a generic Radius connection only when the source parses the exact connection projection supplied by the configured Radius version.

Do not leave a required input implicit because a resource is connected. A direct property or secret reference creates a dependency edge without a connection.

## Process, network, and storage checks

- `containerPort` exposes a network endpoint; it does not change the process listener. Set the app's listener configuration when its default differs.
- Kubernetes `command` replaces the image `ENTRYPOINT`; `args` replaces `CMD`. Preserve the image defaults unless an inspected runtime contract requires an override.
- Before using shell-based runtime composition, confirm the image contains that shell and every invoked binary.
- Ensure config/data paths are writable for the image user. Add persistent storage only when state must survive restarts.
- Keep migrations and verification probes distinct from the long-running application. Use an init role only when the application genuinely requires it.

## Provider compatibility and ownership

Inspect the concrete type, registered recipe, and client source together. Derive FQDN suffixes, TLS, ports, auth modes, connection-string formats, protocol compatibility, network/firewall requirements, and sensitive outputs from that contract. A type named Kafka or RabbitMQ may be backed by a managed service with a compatible surface; the client must support the actual protocol and authentication mode.

`app.bicep` owns developer intent and runtime wiring. Environment/provider Bicep owns recipe modules, SKUs, region, quota, firewall/network policy, and provider-output mapping. Add provider-specific values to the app model only when the application must consume them at runtime.

## Static consistency pass

Before returning the model:

1. Account for every required app-native environment/config input or document an intentional source default.
2. Confirm every secret uses the exact supported secret path and is not exposed through plain state.
3. Confirm every declared port matches a configured process listener.
4. Confirm every command/argument is compatible with the image entrypoint and available binaries.
5. Confirm every writable/persistent path is modeled appropriately.
6. Confirm every connection is consumed by source or intentionally retained for Radius relationship metadata.
7. Confirm the selected recipe/provider protocol and auth contract is compatible with the application client.
