---
name: app-modeling
description: >
  Analyze an application (source repository, source code or description) and generate or
  update its application definition. Use when asked to create or update an
  application definition, model an application, or enumerate an application's
  resources (services, datastores, connections, ingress).
---

# Application modeling

Produce or update an application definition: one file describing the application, its resources 
(services, datastores, ingress, secrets), and their connections. This file is the source of truth
for the definition of an application and its "high-level" infrastructure dependencies. Path: `.radius/app.bicep`.

## Procedure

1. Determine if the repository holds an application that can be deployed (`app-modeling-compatible-applications.md`).
   If the user invokes this skill against an incompatible application or pattern, respond with the appropriate 
   response as defined in `app-modeling-compatible-applications.md`.
2. Read the application code and determine the infrastructure dependencies.
3. Create (or update) `.radius/app.bicep` and add (or update) a comment on the top of the file that explains the
   high-level representation of the app and its dependencies.
4. Update the `.radius/app.bicep` to model the application in Bicep, using the Radius types available to you.
   See `app-modeling-radius-usage.md` for instructions and best practices.
5. Validate that the `.radius/app.bicep` is valid, correct, and deployable using `app-modeling-validation.md`.
   if there was evidence found that the app is not compatible with this skill (for example, has an infrastructure dependency
   not listed in `app-modeling-compatible-applications.md`), then return the appropriate response.
6. Respond with a success message, and show the user the application graph.

## Resources

This workflow is powered by Radius (radapp.io). All information about how to use Radius to define
the application definition is here in this skill and its reference material. See `app-modeling-radius-usage.md`
for additional information.
