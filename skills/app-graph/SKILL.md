---
name: app-graph
description: >
  Parse a .radius/app.bicep file and generate a static application graph
  artifact stored on the radius-graph orphan branch. Enables the Radius
  GitHub Extension to display interactive application topology graphs on PR
  pages and repository tabs. Use when asked to generate an app graph,
  visualize a Radius application, or create the graph artifact.
---

# Radius Application Graph Generation

Use this skill to parse a `.radius/app.bicep` file and produce a `StaticGraphArtifact` JSON
stored on the `radius-graph` orphan branch. The Radius GitHub Extension reads this artifact
to render interactive Cytoscape.js graphs on PR pages and repository tabs.

## Output Format

Your entire visible response must follow this exact sequence. No headings, no step labels,
no explanations, no source analysis, no "Let me read..." preamble.

1. Say: I will generate an application graph for `<app-name>`.
2. Say: First, let me review the graph schema and parsing rules.
3. Show exactly these lines as a single blockquote:
   > Read application graph artifact schema.
   > Read Bicep-to-graph parsing rules.
   > Read resource ID synthesis rules.
4. Say: I see the application `<app-name>` has these resources:
5. List resources as a numbered list (e.g. "1. Applications.Core/applications: todo-list-app")
6. Say: It has these connections:
7. List connections as a numbered list (e.g. "1. todo-list-frontend → mysql (Outbound)")
8. Say: The application graph artifact has been generated.
9. Output the `{branch}/app.json` code block (full JSON).
10. Say: Would you like me to commit this artifact to the `radius-graph` orphan branch?

That is the COMPLETE chat response. Nothing before step 1, nothing after step 10.
Do NOT automatically commit. Wait for the user to confirm.

If the user confirms, commit the artifact to the `radius-graph` orphan branch following
the rules in [committing-the-artifact.md](references/committing-the-artifact.md).

## Internal Workflow (do NOT show these steps to the user)

Internally, before producing the output above:

1. Read `.radius/app.bicep` from the repository.
2. Read [graph-schema.md](references/graph-schema.md) for the `StaticGraphArtifact` structure.
3. Read [bicep-parsing-rules.md](references/bicep-parsing-rules.md) for how to convert
   each Bicep `resource` declaration into an `ApplicationGraphResource`.
4. Read [id-synthesis.md](references/id-synthesis.md) for how to compute synthetic resource IDs.
5. Parse each `resource` block → `ApplicationGraphResource`.
6. Resolve connections: for each `properties.connections` entry, emit an Outbound connection
   on the declaring resource and an Inbound connection on the target resource.
7. Compute `diffHash` for each resource per the rules in [id-synthesis.md](references/id-synthesis.md).
8. Assemble `StaticGraphArtifact` and validate against the checklist below.

Then produce the output in the exact format above.

## Validation Checklist

Before outputting, verify ALL:
- [ ] `version` is `'1.0.0'`
- [ ] `generatedAt` is a valid ISO 8601 timestamp
- [ ] `sourceFile` is `'.radius/app.bicep'`
- [ ] Every Bicep `resource` block produces exactly one `ApplicationGraphResource`
- [ ] `id` for each resource follows the `/planes/radius/local/...` synthesis rule
- [ ] `type` is the full resource type string WITHOUT the API version suffix
- [ ] `provisioningState` is `'Succeeded'` for all resources
- [ ] Each `connections` entry has both `id` (target resource's synthetic ID) and `direction`
- [ ] Both Outbound (on source) and Inbound (on target) connection entries are present
- [ ] `outputResources` is `[]` for all resources (static graph — no live deployment)
- [ ] `diffHash` is set on every resource
- [ ] `appDefinitionLine` is set on every resource (1-based line number in app.bicep)
- [ ] The JSON is valid and well-formed

## Guardrails

- Do NOT include API version in the `type` field.
- Do NOT omit the `Applications.Core/applications` resource — it is always in the graph.
- Do NOT add connections for `param` references (e.g. `environment`) — only `resource` references.
- Do NOT set `outputResources` to anything other than `[]` for static graphs.
- ALWAYS emit both Outbound and Inbound sides of every connection.
- ALWAYS include `appDefinitionLine` so the GitHub Extension can link to source lines.
