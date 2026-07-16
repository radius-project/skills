# Example: Todo-List-App (dockersamples/todo-list-app)

This example records the reasoning and acceptance checks rather than a complete
`app.bicep`, so its resource names and values are not copied into unrelated
applications.

## Selected profile

The requested profile runs the application with MySQL instead of its default
SQLite path. The source supports that profile when `MYSQL_HOST` is present, so
the SQLite default does not override the explicit selection.

| Acceptance criterion | Source/schema-backed decision |
|---|---|
| MySQL backing service | Emit the exact configured `Radius.Data/mySqlDatabases` type |
| Source-built workload | Use the complete Dockerfile context at an immutable source ref |
| Image tag | Set the pinned source commit as the tag because the selected recipe's omitted-tag path is broken |
| Build platform | Override the recipe's multi-platform default with `linux/amd64` |
| Native database contract | Supply `MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASSWORD`, and `MYSQL_DB` |
| Runtime secret | Bind `MYSQL_PASSWORD` through an authored secret and `secretKeyRef` |
| Listener | Expose the source-configured port 3000 |

## Source analysis

- **Role**: long-running Node.js/Express web service
- **Listener**: port 3000, configured by the application
- **Persistence**: SQLite by default; MySQL when `MYSQL_HOST` is present
- **Native configuration read by source**: `MYSQL_HOST`, `MYSQL_USER`,
  `MYSQL_PASSWORD`, `MYSQL_DB`
- **Backing service**: MySQL 8.0 with database `todos`
- **Image**: complete Dockerfile/build context, pinned to an immutable source commit
- **Image tag**: the selected recipe interpolates a null optional tag during
  validation, so use the Docker-valid source commit SHA explicitly
- **Build behavior**: the Dockerfile runs Node, npm, and node-gyp in target-image
  stages and has no `BUILDPLATFORM`/`TARGETARCH` cross-build strategy; the
  containerImages recipe defaults to `linux/amd64` plus `linux/arm64` without
  emulation
- **Storage**: the modeled MySQL service owns persistence; no application
  filesystem volume is required
- **Primary pattern**: Web App

## Modeling decisions

1. The explicit MySQL profile selects the optional source-supported MySQL path;
   do not fall back to SQLite merely because it is the application default.
2. Resolve the MySQL type, API version, credential inputs, and `host` output
   against the exact configured extension and recipe.
3. Map all four native variables. A generic connection does not invent these
   application-specific names.
4. Pass the developer-supplied password to the schema's sensitive resource
   property from `@secure()`, and separately expose it to the workload through a
   supported `Radius.Security/secrets` resource and `secretKeyRef`. Verify that
   the target Environment registers a secrets recipe; a custom pack containing
   only MySQL, containers, and containerImages cannot deploy this model.
5. Referencing the image, MySQL host, and runtime secret creates dependency
   ordering. Omit a generic connection unless the request explicitly requires
   Radius relationship metadata or the source consumes its exact projection.
6. Set `tag` to the pinned source commit because the selected recipe's
   omitted-tag path fails, and set `build.platforms` to `['linux/amd64']` for
   the amd64 target instead of accepting the incompatible multi-platform
   default. Then consume the source build through the image resource's verified
   `properties.imageReference`.
7. Match `containerPort` to the inspected process listener. Do not add a route
   unless external ingress is requested.

## Completion checks

- The selected MySQL type and source-built workload are both emitted.
- The image uses a Docker-valid immutable tag instead of the recipe's broken
  omitted-tag path.
- The image build requests only `linux/amd64`; it does not attempt the
  unsupported arm64 target.
- Every required native variable appears with exact spelling and format.
- The workload password uses `secretKeyRef`; no password is hardcoded or placed
  in a plain environment value.
- The target Environment registers recipes for MySQL, secrets, containerImages,
  and containers.
- The process listener, image entrypoint, and database name/version agree with
  the pinned source.
- The definition compiles against the exact configured extension and has no
  unresolved runtime caveat.
