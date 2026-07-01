# Compatible applications

Reference for `app-modeling.md`. Use this to decide whether a repository holds an
application that Radius can model and deploy, and to choose the correct response when it
does not.

Radius models **long-running, containerized applications** and their "high-level"
infrastructure dependencies (datastores, caches, message brokers, secrets, ingress). It is
not a general-purpose deployment tool for every workload. Model an application only when the
repository fits the compatible patterns below.

## How to decide

Inspect the repository for evidence of a deployable application:

1. **Compute.** Is there a service that runs as a long-lived process and can be packaged as
   a container? Signals: a `Dockerfile`/`Containerfile`, a web/server framework (Express,
   Flask, FastAPI, Spring Boot, ASP.NET, Go `net/http`, Rails), a listening port, or a
   container image referenced in CI. A repo with no containerizable compute is usually not
   modelable.
2. **Dependencies.** Enumerate backing infrastructure the code talks to: databases, caches,
   queues, secrets, object stores. Confirm each maps to a Radius resource type (see
   `app-modeling-radius-usage.md`) or can be created as one (see `app-modeling-type-creation.md`).
3. **Shape.** Prefer one service or a small set of services that communicate over the
   network. Microservices, an API plus a datastore, or a frontend plus backend are ideal.

If the repository has containerizable compute **and** every dependency is supported (or
modelable), it is compatible. Otherwise, return the matching response below.

## Incompatible patterns

| Pattern | Why it is out of scope |
|---|---|
| Library, SDK, CLI tool, or docs-only repository | No deployable application or service |
| Static site with no server (S3 / Netlify / Pages style) | No containerized compute to model |
| Mobile or desktop application | Not deployable infrastructure |
| Serverless functions (AWS Lambda, Azure Functions) as the unit of deploy | Radius targets long-running containers, not function runtimes |
| VM-based workloads or monoliths that require a VM | Radius targets containerized compute |
| Infrastructure dependency with no resource type and no recipe | Cannot be provisioned or connected |

## Responses for incompatible applications

When the application is not compatible, do **not** create or modify `.radius/app.bicep`.
Return the response that matches the situation, adapting the specifics to the repository.

**R1 — Not a deployable application.**
> This repository looks like a `<library / CLI tool / documentation set>` rather than a
> deployable application, so there's nothing for Radius to model. Radius models long-running
> containerized services and their infrastructure dependencies. If you point me at a service
> in this repo (or a repo that builds a container), I can model it.

**R2 — No containerizable compute.**
> I couldn't find a long-running service that can be packaged as a container (no Dockerfile,
> server framework, or listening port). Radius models containerized applications, so there's
> no compute to define here. Add or identify a containerized service and I'll model it.

**R3 — Unsupported infrastructure dependency.**
> This application depends on `<dependency>`, which doesn't have a Radius resource type
> available and isn't something I can model today. I can model the rest of the application,
> but this dependency would be left out. Options: (1) author a resource type for it
> (see `app-modeling-type-creation.md`), (2) substitute a supported equivalent, or
> (3) proceed without it. How would you like to continue?

**R4 — Out-of-scope workload.**
> This looks like a `<serverless / static-site / VM-based>` workload, which is outside what
> Radius models. Radius targets long-running containerized applications. If part of this
> system runs as a container, I can model that portion.

When `app-modeling.md` step 5 (validation) surfaces evidence of incompatibility — for
example a dependency with no resolvable resource type — stop and return the matching
response above instead of producing an invalid definition.
