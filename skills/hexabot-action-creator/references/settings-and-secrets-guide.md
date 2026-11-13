# Settings And Secrets Guide

Use this guide for actions that call external systems, use LLMs, or need runtime configuration.

## Principle

Workflow YAML should reference configuration and credentials, not contain secret values. Action code resolves secrets at runtime and never logs or returns them.

## Credentials

- Prefer Hexabot credential references over raw tokens in input or settings.
- Existing API examples use UI metadata with `AutoCompleteWidget` and `entity: 'Credential'`.
- Resolve values with `context.services.credentials.findOneValue(credentialId)`.
- Treat an empty credential value as a configuration error and throw an actionable message.
- Do not include raw credential values in logs, action outputs, errors returned to workflows, or test snapshots.

Example setting field:

```ts
api_key: z.string().optional().meta({
  title: 'Credential',
  description: 'Credential used to authenticate with the external API.',
  'ui:widget': 'AutoCompleteWidget',
  'ui:options': {
    entity: 'Credential',
    valueKey: 'id',
    labelKey: 'name',
    enableEntityAddButton: true,
  },
});
```

## Environment and config

- Inspect current config and settings services before reading environment variables directly.
- API package guidance prefers central config over direct `process.env` access.
- For external action packages, document required env vars or credential records, but keep workflow YAML secret-free.
- Use settings for non-secret runtime choices such as base URL, mode, region, or response limits.

## External API actions

- Prefer the built-in `http_request` pattern for Node HTTP calls: Axios's HTTP adapter with `timeout: settings.timeout_ms ?? 10000` and the action's runtime `signal`. Standalone action packages must declare Axios as a direct dependency.
- Normalize upstream responses into a stable output shape.
- Include upstream status and safe request IDs when useful.
- Redact headers and payload fields that may contain secrets or PII.
- Use idempotency keys for create/update side effects when supported.
- Decide whether network errors should throw or return `{ success: false, error }` based on whether workflow branching should handle them.

### Timeouts and cancellation

`settings.timeout_ms` controls Hexabot's per-attempt runtime wrapper. Configure the HTTP client as well: a runtime deadline, the client's connection/header/body timers, and server or proxy timeouts are separate limits. Increasing one does not increase the others. Inspect the installed runtime and client versions before assuming their timeout behavior.

- In the bundled CRM example and built-in `http_request`, an omitted timeout uses a 10-second client fallback. Explicit `timeout_ms: 0` disables the runtime wrapper and Axios's timeout, while workflow cancellation still applies.
- Receive `signal` in `execute` and pass it to the HTTP client. Rethrow cancellation errors instead of converting a cancelled action into a normal failure output.
- Axios's Node HTTP adapter avoids Undici's header/body defaults. Selecting Axios's fetch adapter reintroduces fetch transport limits. Client timeouts need not measure the same interval as the runtime deadline; for example, an HTTP socket inactivity timeout is not a total execution budget.
- Node's global fetch uses Undici, whose default header timeout and body inactivity timeout are each 300,000 ms. The body timer measures gaps between chunks, not total response duration. `AbortSignal.timeout(600000)` does not extend these timers; configure a dispatcher for longer requests.
- `AbortSignal.timeout(0)` aborts immediately. To represent Hexabot's disabled timeout with fetch, omit the timeout signal and retain the runtime `signal`.
- For fetch diagnostics, inspect safe fields such as `error.cause.code` (`UND_ERR_HEADERS_TIMEOUT` or `UND_ERR_BODY_TIMEOUT`). A bare `fetch failed` message is insufficient to identify the cause. Log safe codes and configured budgets, never complete client errors or request configuration containing credentials.

In the inspected Hexabot core, `AbstractAction.run` uses `withTimeout` to reject when the budget expires, but that timer does not abort the signal supplied to `execute`. A previous attempt can keep running when a retry starts. Forwarding `signal` handles workflow cancellation but does not repair this core timeout behavior. Configure client timeouts and disable retries when repeating the operation could duplicate effects.

Sources: [core HTTP action](https://github.com/hexabot-ai/Hexabot/blob/main/packages/api/src/extensions/actions/web/http-request.action.ts), [action runtime](https://github.com/hexabot-ai/Hexabot/blob/main/packages/agentic/src/action/abstract-action.ts), [timeout wrapper](https://github.com/hexabot-ai/Hexabot/blob/main/packages/agentic/src/utils/timeout.ts), [Undici client timers](https://undici.nodejs.org/api/Client), and [Node custom dispatchers](https://nodejs.org/api/globals.html#custom-dispatcher).

### Fetch alternative

When fetch is appropriate, use a compatible Undici dependency and configure its dispatcher explicitly. Place the import at module scope and the remaining code inside the action's `execute`; `endpoint`, `settings`, and the runtime `signal` come from that action. Consume the body before closing the dispatcher.

```ts
import { Agent, fetch } from 'undici';

const timeoutMs = settings.timeout_ms ?? 10000;
const dispatcher = new Agent({
  headersTimeout: timeoutMs,
  bodyTimeout: timeoutMs,
});
const requestSignal =
  timeoutMs > 0
    ? AbortSignal.any([signal, AbortSignal.timeout(timeoutMs)])
    : signal;

try {
  const response = await fetch(endpoint, {
    dispatcher,
    signal: requestSignal,
  });
  const body = await response.text();
  // Check response.ok and normalize body into the action's output contract.
} finally {
  await dispatcher.close();
}
```

This action owns its per-call dispatcher and closes it on success or failure. If a service owns a shared dispatcher, close it only at service shutdown, not after each request. Consume or cancel every response body; use `destroy()` when abandoning outstanding work. Avoid changing the process-wide dispatcher from an individual action. Disabling header/body timers leaves other limits, including connection and proxy timeouts, in effect.

### Long-running requests and retry safety

Choose an external call's budget from its expected latency and the service's limits. This illustrative HTTP operation allows ten minutes and explicitly disables retries because repeating it may have side effects, overriding any inherited retry policy:

```yaml
defs:
  external_request:
    kind: task
    action: http_request
    inputs:
      url: https://api.example.com/operations
      body: >-
        {"operation":"process","resource_id":"example"}
    settings:
      method: POST
      timeout_ms: 600000
      retries:
        enabled: false
flow:
  - do: external_request
outputs:
  result: =$output.external_request
```

Replace the URL, payload, and timeout with the target service's contract and expected latency. Align server and proxy limits with the expected wait; a shorter proxy timeout can still terminate the request. `http_request` returns ordinary network and non-2xx failures as outputs; inspect `ok` before consuming `body` as a successful result.

A timeout or disconnect leaves the remote outcome uncertain: the operation may still be running or may already have completed. Determine retry safety from its effects, even if the API uses GET. Repeated calls must be safe through the operation's own idempotence or upstream-enforced deduplication; sending an idempotency header alone is insufficient. If a service supports job IDs and status polling, prefer observing the existing job over starting it again. These considerations apply to report generation, bulk updates, crawling, and other external operations.

Hexabot retries thrown failures, including runtime timeout errors, when the effective retry policy permits it. Returning `{ success: false }` or `{ ok: false }` is a normal action result and does not trigger automatic retries; workflow logic must handle it. Cancellation should propagate by throwing. See the [failure and retry policy](action-contract-guide.md#failure-and-retry-policy).

## LLM actions

- Prefer existing AI base actions and schemas before creating new LLM action logic.
- Use `model` bindings for model provider, model ID, base URL, organization, and credential reference when following current built-in AI patterns.
- Do not put provider API keys directly in action inputs, outputs, or workflow YAML.
- Be explicit about prompt source, memory usage, tool access, output schema, and model settings.
- Avoid passing secrets or unnecessary PII into prompts.

## Memory actions

- Use `context.memoryStore` for workflow memory reads/writes.
- Validate memory slugs and fields against current memory definitions when the repo exposes that information.
- Return the updated or read memory values in a stable shape.
- For AI memory bindings, inspect current `memory.binding.ts` and AI base action behavior before generating code.

## Logging

- Use `context.services.logger`.
- Log action name, target system, status, safe IDs, and useful error summaries.
- Do not use `console`.
- Never log raw tokens, authorization headers, credential values, complete customer records, or full model prompts unless the existing observability policy explicitly allows it.
