# Hexabot Action Creator Skill

This repository packages a skill that helps AI coding agents design, scaffold, implement, review, and document Hexabot v3 actions.

The agent-facing instructions live in [`skills/hexabot-action-creator/SKILL.md`](skills/hexabot-action-creator/SKILL.md). This README is human-facing GitHub documentation for the skill repository.

## What Hexabot Is

Hexabot v3 is a self-hostable AI automation platform for building workflows that talk, act, and remember. It combines workflow definitions, schema-validated actions, agentic behavior, conversational channels, reusable bindings, memory and RAG capabilities, MCP integration points, and human handoff patterns in one runtime.

Teams can use Hexabot to build customer-facing assistants, internal service automations, scheduled workflows, integration agents, and operational AI systems while keeping the platform and production data under their own control.

## What This Skill Provides

- Action authoring guidance for Hexabot API actions, custom application actions, npm-distributed `hexabot-action-*` packages, and standalone `@hexabot-ai/agentic` examples.
- Pattern selection guidance for `createAction`, `BaseAction`, and `defineAction`.
- Contract design checklists for action names, workflow types, input schemas, output schemas, settings schemas, supported bindings, idempotency, and failure behavior.
- Settings and secrets guidance for credentials, environment variables, external APIs, LLM model bindings, memory, and MCP integration points.
- Review and validation checklists for parsing, schemas, output shape, error paths, idempotency, logging, tests, and binding compatibility.
- Output templates for polished action designs, scaffolds, reviews, and documentation responses.
- An illustrative TypeScript action example for creating a CRM ticket.

## Repository Layout

```text
skills/hexabot-action-creator/
+-- SKILL.md
+-- agents/
|   `-- openai.yaml
+-- examples/
|   `-- create-crm-ticket.action.example.ts
`-- references/
    +-- action-authoring-guide.md
    +-- action-contract-guide.md
    +-- output-templates.md
    +-- settings-and-secrets-guide.md
    `-- testing-and-review-checklist.md
```

## How To Use

Ask the agent to use this skill when you need to:

- Convert a business integration requirement into a Hexabot action contract.
- Implement a custom Hexabot workflow action.
- Scaffold or improve a `hexabot-action-*` package.
- Wrap an external API, LLM call, memory operation, MCP server, messaging capability, or internal service as an action.
- Review an existing action for schema quality, runtime compatibility, settings and credential handling, tests, and workflow ergonomics.
- Produce developer-facing documentation for a Hexabot action or action package.

For the most accurate output, provide the target Hexabot repository or action package, the intended workflow type, required inputs and outputs, configured bindings, credential model, external API details, and whether runtime verification is available.

## Validation Notes

The skill is designed to prefer live source-of-truth checks before final code or review findings. When available, the agent should inspect the active Hexabot monorepo, generated application, installed Hexabot packages, or standalone action package before choosing decorators, imports, schema fields, binding kinds, registration hooks, package layout, or config conventions.

If only this packaged skill is available, the included references and example can guide design and drafting, but runtime verification still requires a checked-out Hexabot project, compiled action package, or running Hexabot environment.

## Useful Hexabot Links

- Website: <https://hexabot.ai/>
- Documentation: <https://docs.hexabot.ai/>
- GitHub repository: <https://github.com/hexabot-ai/Hexabot>
- GitHub issues: <https://github.com/hexabot-ai/Hexabot/issues>
- GitHub pull requests: <https://github.com/hexabot-ai/Hexabot/pulls>
- CLI package: <https://www.npmjs.com/package/@hexabot-ai/cli>
- CLI source docs: <https://github.com/hexabot-ai/Hexabot/tree/main/packages/cli>
- API action source: <https://github.com/hexabot-ai/Hexabot/tree/main/packages/api/src/actions>
- Built-in action examples: <https://github.com/hexabot-ai/Hexabot/tree/main/packages/api/src/extensions/actions>
- Extensions marketplace: <https://hexabot.ai/extensions>
- Discord community: <https://discord.gg/hexabot>

## Local Development Pointers

When authoring actions against a local Hexabot project, the default development endpoints are:

- Admin UI: <http://localhost:3000>
- API: <http://localhost:3000/api>
- API docs, when enabled in non-production environments: <http://localhost:3000/docs>

Common project bootstrap commands:

```bash
npm install -g @hexabot-ai/cli
hexabot create my-project
cd my-project
hexabot dev
```

Inside the Hexabot monorepo, action-related source usually starts in:

- `packages/api/src/actions/**`
- `packages/api/src/extensions/actions/**`
- `packages/api/src/bindings/**`
- `packages/agentic/src/action/**`

## License And Attribution

This skill is documentation and guidance for working with Hexabot actions. Refer to the official Hexabot repository for current Hexabot license terms, attribution requirements, and contribution guidelines.
