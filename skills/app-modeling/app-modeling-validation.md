# Validating the application definition

Reference for `app-modeling.md` step 5. Use this to confirm `.radius/app.bicep` is valid and
correct before reporting success.

This skill runs in an environment **without** the Radius or Bicep command-line tools, so
validation is performed by **static review** of the definition — reading the file and checking
it against the resource type schemas (the `<type>.yaml` manifest for each type). The goals are
to confirm the definition is structurally well-formed, every resource type and `apiVersion`
resolves to a known type, and
the model is correct (every resource is wired to the app and the environment, and connections
are sound). If validation surfaces evidence that the app is not compatible — for example a
dependency with no resolvable resource type — stop and return the matching response from
`app-modeling-compatible-applications.md`.

## Project configuration: bicepconfig.json

A well-formed Radius project includes a `bicepconfig.json` at its root that declares the Radius
Bicep extension. This is what lets the `Radius.*` types resolve. Ensure one exists, and create
it if it is missing:

```json
{
  "experimentalFeaturesEnabled": {
    "extensibility": true
  },
  "extensions": {
    "radius": "br:biceptypes.azurecr.io/radius:0.59"
  }
}
```

Set the `radius` extension version to match the Radius release the project targets (the example
tag above is illustrative).

## Validate the definition

No compiler is available in this environment, so validate by static review. For each resource
in `.radius/app.bicep`:

- Confirm the `<namespace>/<type>@<apiVersion>` resolves to a known type — a default type or
  another registered Radius resource type (see the catalog in
  `app-modeling-available-resource-types.md`).
- Check every property against that type's schema: required properties are present, names and
  types match, and `readOnly` outputs (such as `host`, `port`, `imageReference`) are produced by
  the resource rather than set as inputs.
- Trace each `connections.<name>.source` to a resource declared in the same file.

Then work through the correctness checklist below. A definition that passes these checks is
ready to hand off for deployment, which happens in the Radius control plane outside this
environment.

## Correctness checklist

Confirm:

- [ ] The file begins with `extension radius`.
- [ ] `param environment string` exists and every resource sets
      `properties.environment: environment`.
- [ ] Exactly one `Radius.Core/applications` resource exists; every other resource sets
      `properties.application: app.id`.
- [ ] Every resource type and `apiVersion` matches a known type (a default type or another
      registered Radius resource type).
- [ ] Every `connections.<name>.source` points at the `.id` of a resource declared in the
      file.
- [ ] All required properties for each type are set (e.g. PostgreSQL requires `environment`
      and `secretName`; `Radius.Compute/containerImages` requires `build.source`).
- [ ] Images built from source use `Radius.Compute/containerImages`, and the container's
      `image` references the build's `properties.imageReference`.
- [ ] No secrets are hard-coded; sensitive values come from `Radius.Security/secrets` and
      `@secure()` parameters.
- [ ] The top-of-file comment accurately describes the app and its dependencies.

## Common errors and fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `Unable to resolve the module ... extension` / unknown type | Missing or wrong `bicepconfig.json`, or wrong extension version | Add/repair `bicepconfig.json`; align the `radius` extension version |
| Type `Radius.X/y` not found | Not a default type and not registered | Author/register the type (see `app-modeling-type-creation.md`) or pick a different type |
| Wrong API version | apiVersion does not match the type manifest | Use the `apiVersion` from the type's `.yaml` manifest |
| Missing required property | Required schema property omitted | Add it (consult the type manifest's `required` list) |
| Dangling connection | `source` references an undeclared or misnamed resource | Point `source` at an existing resource's `.id` |
| Plaintext secret in `env` or properties | Sensitive value inlined | Move to `Radius.Security/secrets` + `@secure()` parameter |

## When validation reveals incompatibility

If a dependency cannot be resolved to any resource type and cannot reasonably be authored as
one, the application is not fully modelable. Do not ship an invalid or partial definition
silently — return the unsupported-dependency response (R3) or the appropriate response from
`app-modeling-compatible-applications.md`.
