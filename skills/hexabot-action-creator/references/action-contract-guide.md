# Action Contract Guide

Design the action contract before writing code. The contract should be stable enough for workflow YAML authors to rely on.

## Contract checklist

- Action name: `snake_case`, usually verb-object, for example `create_ticket`, `update_memory`, `retrieve_rag_content`.
- Purpose: one bounded capability and explicit side effects.
- Workflow types: all by default, or a restricted list from `WorkflowType`.
- Input schema: business payload evaluated from YAML literals or JSONata.
- Output schema: stable JSON-serializable result referenced through `$output.<task>`.
- Settings schema: execution configuration, not per-call business payload.
- Supported bindings: only binding kinds the action actually consumes.
- Failure behavior: throw, retry, or return an explicit failure output.

## Input schemas

- Use Zod schemas and infer TypeScript types with `z.infer`.
- Prefer strict validation for externally meaningful contracts.
- Use `.meta({ title, description })` for editor-friendly fields.
- Put payload fields in `inputs`, not `settings`.
- Validate IDs, URLs, emails, enums, minimum lengths, and required combinations with `.superRefine()` when needed.

## Output schemas

- Return plain objects, arrays, strings, numbers, booleans, or null-compatible data.
- Include stable external identifiers such as `ticket_id`, `request_id`, or `status`.
- Prefer explicit success/failure unions when workflows should branch on the result.
- Throw instead of returning failure when configuration is invalid or the workflow cannot reasonably recover.
- Do not expose raw credential values, bearer tokens, or full sensitive upstream payloads.

## Settings schemas

- Use settings for execution knobs: API base URL, credential reference, timeout-sensitive mode, method, model parameters, or behavior toggles.
- Do not redefine base settings keys from `@hexabot-ai/agentic`: `timeout_ms` and `retries`.
- Base settings are parsed together with action settings and are available during execution through `settings.timeout_ms` and `settings.retries`.
- Use `z.strictObject()` for action settings unless the existing local pattern uses a looser schema.
- Use defaults for safe, unsurprising behavior.

## Bindings

- Declare `supportedBindings` only when the action consumes runtime bindings.
- Built-in AI actions commonly support `tools`, `mcp`, `model`, and `memory`.
- `model` is a single binding kind; `tools`, `memory`, and `mcp` are multiple binding kinds in the current API.
- If adding a new binding kind inside the Hexabot monorepo, inspect `packages/api/src/bindings/**` and follow `createBindingKind` examples. In a standalone package, inspect the installed API exports or target package template first. Do not invent binding metadata.
- Validate nested binding support when a binding definition mounts other bindings.

## Workflow compatibility

- A task def calls an action with:

```yaml
defs:
  create_ticket:
    kind: task
    action: create_crm_ticket
    inputs:
      subject: =$input.subject
```

- Task inputs are evaluated before the action runs.
- The raw action output is stored under `$output.<task>`.
- Task-level output mapping is not supported in the current DSL.
- Workflow metadata such as name, schedule, and type lives outside the YAML definition.

## Failure and retry policy

- Choose retries from the action's effects and repeatability. An action name or HTTP method alone does not establish retry safety; confirm that reads are repeatable too.
- For actions with side effects, establish idempotence or upstream-enforced deduplication with a stable operation key across attempts. Merely accepting or sending an idempotency key does not establish retry safety.
- When repeating an operation could duplicate effects, explicitly set task-level `retries.enabled: false` to override any inherited retry policy. A timeout or disconnect leaves the remote outcome uncertain.
- Thrown errors and runtime timeouts can trigger retries under the effective policy. Returned failure outputs such as `{ success: false }` or `{ ok: false }` do not; workflows must branch on them.
- Forward the runtime `signal` to external calls and rethrow cancellations. The inspected runtime timeout wrapper rejects without aborting the running attempt; see [timeouts and cancellation](settings-and-secrets-guide.md#timeouts-and-cancellation) before assuming retries cannot overlap.
- Log safe correlation data and return safe failure fields that workflows can branch on.
