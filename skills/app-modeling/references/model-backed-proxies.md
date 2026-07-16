# Model-Backed Proxies

Model gateways and proxies need more than an endpoint and API key. Their configuration must identify the exact provider deployment that the selected Recipe creates. Treat a workload as a model gateway or proxy when it re-exposes model access or aliases to other clients, or rewrites client requests into provider-specific routing.

A runnable default profile must configure at least one model route. When the request does not select a provider, use a compatible `Radius.AI/models` type and the target Environment's concrete default Recipe: take the provisioned model identifier from the exact schema default or another source-compatible schema value, and take the provider deployment identifier from the Recipe. Do not omit the model resource merely because a compose example leaves provider credentials to the developer. Do not substitute an optional gateway database for the inference route.

## Resolve distinct identifiers

Keep these values separate in the requirement ledger:

| Value | Evidence |
|---|---|
| Application-facing alias | The model name clients send to the proxy |
| Radius resource name | The `Radius.AI/models` resource instance name |
| Provisioned model identifier | The schema input consumed by the Recipe, such as a model family or SKU |
| Provider deployment identifier | The verbatim Recipe parameter or resource line that names the concrete deployment |
| Provider prefix | The source-supported provider selector, such as an Azure-specific prefix |

Never assume the provider deployment identifier equals the Radius resource name, application alias, or `properties.model`. Inspect the exact Recipe parameter mapping. For example, a Recipe can use `properties.model` as the Azure OpenAI model name while creating a deployment with a separate fixed name; the proxy must address that deployment name.

A provider deployment identifier may be a fixed Recipe literal rather than a resource schema property or Recipe output. Cite that verbatim Recipe line as the ledger evidence and use the literal in the provider configuration. This does not relax schema validation for resource property reads: do not invent a resource output for the literal or substitute a schema property with a similar name.

## Resolve endpoint, version, and authentication

Prove all of the following together:

1. The resource schema exposes the endpoint path the workload references.
2. The selected Recipe maps its concrete endpoint output to that path.
3. The application's provider adapter accepts the endpoint format without an undocumented suffix or path rewrite.
4. The API or protocol version is supported by the pinned application source and the concrete provider deployment.
5. The Recipe maps its credential into an exact managed-secret key.
6. The workload binds that key with `secretKeyRef` from the schema-declared managed-secret name.

A Radius connection projects only the contract supported by the configured runtime. It is not a fallback for a missing managed-secret path, and it must never be assumed to inject a Recipe secret. If the configured extension lacks the managed-secret contract that the target Environment uses, report version drift and stop rather than switching the config to a guessed `CONNECTION_*` secret variable.

## Build runnable proxy configuration

The proxy configuration must contain:

- one usable application-facing alias;
- the exact provider deployment identifier and any provider prefix the selected adapter requires;
- the verified endpoint variable;
- the verified credential variable;
- the verified API or protocol version; and
- any production authentication key the proxy requires, supplied through a separate secret binding.

Prefer a source-packaged configuration. If none exists and the image has a verified shell, executable, and writable path, generate the template at startup and execute the proxy with that file. When the source supports environment references, keep credentials out of the file and bind them separately through `secretKeyRef`. When the source requires credentials in the file, compose it at runtime from secret-bound variables only after verifying expansion, file permissions, escaping, and execution behavior. Do not put a nonsecret template in a secret-backed volume merely to mount it.

## Readiness proof

Compilation, pod readiness, and a health endpoint do not prove model routing. The static consistency pass must cite the verbatim Recipe deployment-name evidence, establish that an authenticated request using the application-facing alias reaches that exact provider deployment, and confirm that the selected request parameters are supported by that model/API version.
