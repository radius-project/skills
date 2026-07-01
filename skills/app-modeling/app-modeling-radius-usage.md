# Modeling applications with Radius

Reference for `app-modeling.md` step 4. Instructions and best practices for writing the
application definition in `.radius/app.bicep` using Radius resource types.

The definition is a Bicep file describing the application, its resources (services,
datastores, ingress, secrets), and the connections between them. Keep it
application-oriented: describe *what* the app needs, not platform-specific infrastructure
detail. How a resource is provisioned is the job of a Recipe on the environment, not the
app definition.

## File structure

Every `.radius/app.bicep` follows the same shape:

```bicep
extension radius

@description('The Radius Environment ID. Supplied by the Radius environment at deploy time.')
param environment string

resource app 'Radius.Core/applications@2025-08-01-preview' = {
  name: 'myapp'
  properties: {
    environment: environment
  }
}

resource apiImage 'Radius.Compute/containerImages@2025-08-01-preview' = {
  name: 'apiImage'
  properties: {
    environment: environment
    application: app.id
    build: {
      source: 'git::https://github.com/example/myapp.git//api?ref=v1.0.0'
    }
  }
}

resource api 'Radius.Compute/containers@2025-08-01-preview' = {
  name: 'api'
  properties: {
    environment: environment
    application: app.id
    containers: {
      api: {
        image: apiImage.properties.imageReference
        ports: {
          web: { containerPort: 8080 }
        }
      }
    }
  }
}
```

Notes:
- Begin the file with `extension radius` (this loads the Radius Bicep type system).
- `param environment string` is required and supplied by the CLI at deploy time. Reference
  it from every resource's `properties.environment`.
- Declare the `Radius.Core/applications` resource and wire every other resource to it via
  `properties.application: app.id`.
- Build images from source with `Radius.Compute/containerImages` and consume the result via
  `properties.imageReference` on a container; see "Building images from source".
- Add a top-of-file comment summarizing the app and its dependencies, per `app-modeling.md`
  step 3.

A `bicepconfig.json` at the project root registers the Radius extension; see
`app-modeling-validation.md`.

## Core resource types

| Need | Resource type |
|---|---|
| The application itself | `Radius.Core/applications` |
| A containerized service | `Radius.Compute/containers` |
| Build a container image from source | `Radius.Compute/containerImages` |
| HTTP ingress / routing | `Radius.Compute/routes` |
| Persistent storage | `Radius.Compute/persistentVolumes` |
| Secrets / credentials | `Radius.Security/secrets` |

## Building images from source

When modeling an application from its source repository, build the container image from
source with `Radius.Compute/containerImages` rather than referencing a pre-built image. The
build runs inside the Radius control plane and pushes to the environment's registry; the
resource exposes a read-only `imageReference` that a container consumes.

```bicep
resource apiImage 'Radius.Compute/containerImages@2025-08-01-preview' = {
  name: 'apiImage'
  properties: {
    environment: environment
    application: app.id
    tag: 'v1.0.0'
    build: {
      source: 'git::https://github.com/example/myapp.git//api?ref=v1.0.0'
      dockerfile: 'Dockerfile'
    }
  }
}
```

- `build.source` (required) — a `git::https://...` URL (BuildKit clones it inside the cluster)
  or a local path to the build context. For git, select a subdirectory with `//<subdir>` and a
  ref with `?ref=<branch-or-sha>`. Pin to a commit SHA or immutable tag so the build is
  content-addressable.
- `build.dockerfile` (optional) — path to the Dockerfile relative to the source. Defaults to
  `Dockerfile`.
- `build.platforms` (optional) — target platforms, e.g. `['linux/amd64', 'linux/arm64']` (the
  default). Multi-arch builds require a cross-compiling Dockerfile (`FROM --platform=$BUILDPLATFORM`,
  `TARGETARCH`).
- `build.args` (optional) — map of `--build-arg` values.
- `tag` (optional) — image tag; when unset the recipe computes a content-addressable digest.
- `imageReference` (read only) — the produced `<registry>/<name>:<tag>`; reference it from a
  container's `image` property.

If the repository already publishes a pre-built image, reference that image tag directly from
the container instead of building from source.

## Containers

A `Radius.Compute/containers` resource holds one or more containers in its `containers` map.
Each container takes an `image` — a built image's `imageReference` (see Building images from
source) or a pre-built tag — and optional `ports`, `env`, `command`/`args`, and `volumeMounts`.

```bicep
resource frontend 'Radius.Compute/containers@2025-08-01-preview' = {
  name: 'frontend'
  properties: {
    environment: environment
    application: app.id
    containers: {
      frontend: {
        image: frontendImage.properties.imageReference
        ports: {
          web: { containerPort: 3000 }
        }
        env: {
          LOG_LEVEL: { value: 'info' }
        }
      }
    }
    connections: {
      db: { source: postgresql.id }
    }
  }
}
```

Here `frontendImage` is a `Radius.Compute/containerImages` resource that builds the image from
source (see Building images from source). Set `image` to a pre-built tag instead if the
repository publishes one.

## Datastores and secrets

Datastores that need credentials pair with a `Radius.Security/secrets` resource. Create the
secret, reference it from the datastore by name, and pass sensitive values as parameters:

```bicep
@secure()
param dbPassword string

resource dbCredentials 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'db-creds'
  properties: {
    environment: environment
    application: app.id
    data: {
      username: { value: 'admin' }
      password: { value: dbPassword }
    }
  }
}

resource postgresql 'Radius.Data/postgreSqlDatabases@2025-08-01-preview' = {
  name: 'postgresql'
  properties: {
    environment: environment
    application: app.id
    size: 'S'
    secretName: dbCredentials.name
  }
}
```

Declare sensitive values as `@secure()` parameters so they are supplied at deploy time rather
than hard-coded in the definition.

## Ingress

Expose an HTTP service to the outside world with `Radius.Compute/routes`, pointing at a
container's exposed port. Internal service-to-service traffic does not need a route — use a
`connection` instead.

## Resolving resource types

When the application needs a dependency:

1. Check the default types above first.
2. Consult the dependency catalog in `app-modeling-available-resource-types.md`, which maps
   common dependencies (databases, caches, object storage, messaging, AI) to Radius types and
   their cloud backends, and notes each type's availability.
3. If not a default, use any other registered Radius resource type whose schema fits the
   dependency. Each type is defined by a `<type>.yaml` manifest whose `description` and property
   docs explain how to use it; use the exact `namespace`, type name, and `apiVersion` from that
   manifest.
4. If no type fits, follow `app-modeling-type-creation.md`, or return the unsupported
   dependency response from `app-modeling-compatible-applications.md`.

## Best practices

- One application resource; wire every resource to it with `application: app.id`.
- Always set `properties.environment` to the `environment` parameter.
- Never hard-code secrets. Use `Radius.Security/secrets` and `@secure()` parameters.
- Keep the definition application-focused; leave provisioning detail to Recipes.
- Use only resource types you can resolve to a real manifest, so the definition validates
  (see `app-modeling-validation.md`).
