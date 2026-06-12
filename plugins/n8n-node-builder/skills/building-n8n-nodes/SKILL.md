---
name: building-n8n-nodes
description: "Builds custom community nodes for n8n, the workflow automation platform. Activates when the user wants to create, scaffold, develop, validate, test, lint, or publish an n8n node — including declarative (REST API), programmatic, and AI sub-node (chat model/memory) styles. Also triggers when the user mentions n8n nodes, n8n-cli, n8n-node, community nodes, node credentials, node linting, cloud-support, the community package verification scan, or anything related to extending n8n with custom integrations. Encodes the official n8n-node CLI toolchain, all current best practices, and a gate-based self-validation protocol for proving a node works."
---

# n8n Node Builder

Build production-ready custom nodes for n8n using the official `n8n-node` CLI tool and n8n's best practices — and prove they work with the validation gate protocol.

## When You Need More Detail

This skill uses progressive disclosure. The SKILL.md covers the full workflow and decision-making. For complete code templates and protocols, read these reference files:

- `references/declarative-node.md` — Full declarative node template with routing, credentials, codex, and advanced patterns (preSend/postReceive, pagination, resourceLocator/resourceMapper, customOperations, hints)
- `references/programmatic-node.md` — Full programmatic node template with execute method, error handling, item linking, trigger patterns, and versioning
- `references/credentials.md` — All credential/auth patterns (API key, Bearer, OAuth2, Basic, Custom, testedBy, preAuthentication)
- `references/validation.md` — The agent self-validation gate protocol: lint rule catalog, build, runtime testing, cloud-support, release prechecks, verification scan
- `references/ai-nodes.md` — AI sub-nodes: chat model and memory nodes via supplyData() and @n8n/ai-node-sdk
- `references/publishing.md` — Releasing, GitHub Actions publishing with provenance, and the verification checklist
- `references/common-mistakes.md` — Error catalog of numbered mistake patterns and fixes

Read the appropriate reference file before writing any code.

## Workflow Overview

Building an n8n node follows this sequence:

1. **Decide** on the node style (declarative vs programmatic vs AI sub-node)
2. **Scaffold** the project with the `n8n-node` CLI
3. **Implement** the node base file, credentials file, codex file, and icon
4. **Validate** with the gate protocol (lint → build → cloud-support → runtime)
5. **Release** via `n8n-node release` and GitHub Actions, then submit for verification

## Step 1: Choose Your Node Style

n8n has two main node-building styles plus the AI sub-node paradigm. Picking the right one up front saves significant rework.

### Declarative Style (preferred for REST APIs)

Use declarative when the integration is a REST API wrapper. It's JSON-based, simpler, more future-proof, and faster to get approved for n8n Cloud.

The declarative style handles data flow through a `routing` key inside the operations object. There's no `execute()` method — n8n constructs HTTP requests from the JSON description automatically. Declarative nodes go far beyond simple routing: `preSend`/`postReceive` transform functions, three pagination modes, declarative dynamic dropdowns, `resourceLocator`/`resourceMapper`/`fixedCollection` parameters, advanced `displayOptions` conditions, and binary file handling are all available. See `references/declarative-node.md` → "Advanced Declarative Patterns" for the complete templates and a TypeScript type reference.

**Choose declarative when:**
- The API is REST-based
- You want a simpler, lower-risk codebase
- Even if you need custom request/response transformation (use `preSend`/`postReceive` functions)
- Even if you need file upload/download (use `preSend` with the global `FormData`, `postReceive` with binary handling)
- Even if you need dynamic field mapping (use `resourceMapper`)
- Even if you need custom pagination logic (use custom pagination functions)
- Even if one or two operations need full programmatic control — use `customOperations` to implement just those resource/operation pairs as programmatic functions while routing handles the rest (a node must not define both `execute()` and `customOperations`)

### Programmatic Style (required for advanced use cases)

Use programmatic when you need full control over execution. It requires an `execute()` method that reads inputs, builds requests, and returns results manually.

**You must use programmatic for:**
- Trigger nodes (webhook, polling, or other event-driven)
- GraphQL APIs
- Non-REST protocols
- Complex multi-step logic that chains multiple sequential API calls where later calls depend on earlier results
- Full versioning (separate implementations per major version via `VersionedNodeType`) — declarative nodes support only light versioning, so choose programmatic if you anticipate breaking-change versions

### AI Sub-Nodes (chat models, memory)

If the goal is to plug a model or memory provider INTO the AI Agent (rather than exposing API operations the agent can call as tools), build an AI sub-node: `supplyData()` instead of `execute()`, AI connection types instead of Main, and the `@n8n/ai-node-sdk`. Read `references/ai-nodes.md`. For "let the agent use my node as a tool", a regular node with `usableAsTool: true` is all you need.

### Quick Decision

Ask: "Is this a REST API with no triggers and no multi-call chaining?" If yes → declarative (even for complex transformation, pagination, file handling, and field mapping — and `customOperations` covers isolated exceptions). Building a model/memory provider for AI Agents → AI sub-node. Otherwise → programmatic.

## Step 2: Scaffold with the n8n-node CLI

The CLI sets up the correct project structure, dependencies, linter config, and build scripts automatically.

### Option A: Without installing (recommended)

```bash
npm create @n8n/node@latest n8n-nodes-<YOUR_NODE_NAME> -- --template <template>
```

### Option B: Install globally

```bash
npm install --global @n8n/node-cli
n8n-node new n8n-nodes-<YOUR_NODE_NAME> --template <template>
```

Templates:
- `declarative/github-issues` — Demo with multiple operations and credentials (good for learning)
- `declarative/custom` — Blank declarative starting point (prompts interactively for base URL and auth type)
- `programmatic/example` — Programmatic with full flexibility
- `programmatic/openai-chat-model` — AI sub-node: chat model for OpenAI-compatible APIs
- `programmatic/custom-chat-model` — AI sub-node: chat model for custom protocols (skeleton)
- `programmatic/custom-chat-model-example` — AI sub-node: custom chat model with worked example
- `programmatic/custom-chat-memory` — AI sub-node: conversation memory

Useful flags for non-interactive (agent) runs: `--skip-install` (skip dependency installation) and `--force` (overwrite an existing directory). Note that `declarative/custom` still prompts for base URL and auth type — prefer `declarative/github-issues` or `programmatic/example` when running fully unattended, or answer the prompts.

### Option C: Clone the n8n-nodes-starter repo

```bash
git clone https://github.com/n8n-io/n8n-nodes-starter.git n8n-nodes-<YOUR_NODE_NAME>
cd n8n-nodes-<YOUR_NODE_NAME>
rm -rf .git && git init
npm install
```

The starter provides pre-configured TypeScript, ESLint, build scripts, and example files. After cloning, rename/replace the example node and credential files with your own and update `package.json`.

### Naming Rules

Package names must follow one of these formats:
- `n8n-nodes-<NAME>` (e.g., `n8n-nodes-acme`)
- `@<ORG>/n8n-nodes-<NAME>` (e.g., `@myorg/n8n-nodes-acme`)

After scaffolding, the project looks like:

```
n8n-nodes-<name>/
├── package.json          # Must contain "n8n" attribute listing nodes and credentials
├── tsconfig.json
├── eslint.config.mjs     # Don't edit — two-line flat config importing from '@n8n/node-cli/eslint';
│                         # with "n8n": { "strict": true }, lint fails if this file is modified
├── .github/workflows/    # ci.yml (lint + build on push) and publish.yml (npm publish with provenance)
├── nodes/
│   └── <NodeName>/
│       ├── <NodeName>.node.ts      # Base file — the node's core logic
│       ├── <NodeName>.node.json    # Codex file — metadata for n8n's node panel
│       └── <NodeName>.svg          # Icon — square SVG (required by lint)
├── credentials/
│   └── <NodeName>Api.credentials.ts  # Credential file
└── dist/                 # Built output (generated by npm run build = n8n-node build)
```

## Step 3: Implement the Node

Every node needs three files at minimum: the base file, the codex file, and the credentials file (unless no auth is needed).

### 3A: The Node Base File (`<Name>.node.ts`)

This is the heart of the node. It exports a class implementing `INodeType` with a `description` object.

**Critical rules:**
- The class name must match the filename (e.g., class `Acme` → file `Acme.node.ts`) — lint enforces this
- Use `NodeConnectionTypes.Main` for inputs/outputs — the **plural** const object, imported as a value from `n8n-workflow` (`import { NodeConnectionTypes } from 'n8n-workflow'`). The singular `NodeConnectionType` is type-only and cannot be used as a value. Never use the string `'main'` — it fails the `node-connection-type-literal` lint rule
- The `name` field in the description must be a camelCase unique identifier
- Use Title Case for `displayName` and all UI-facing strings
- Always set `noDataExpression: true` on Resource and Operation selectors
- Always include `action` on every operation option (e.g., `action: 'Create a contact'`)
- Use `import type` for symbols only used in type annotations (rule of thumb: if a symbol only appears in `: Type` annotations or `as Type` casts, use `import type`; if it's used as a value like `throw new NodeApiError(...)` or `NodeConnectionTypes.Main`, use a regular import)
- Dynamic expressions in routing must start with `=` prefix: `'=/contacts/{{$parameter["id"]}}'`
- **Use routing OR `execute()`, never both** — if a node defines `execute()`, n8n calls it and silently ignores the declarative routing/`requestDefaults` config (the routing engine is only used when the node has no `execute`/`supplyData`/`poll`/`trigger` method)
- The `execute()` method must return `[returnData]` — an array of arrays (one per output connector). Forgetting the outer array is a common error

**Standard description parameters** (same for both styles):

| Parameter | Type | Purpose |
|-----------|------|---------|
| `displayName` | string | Name shown in the UI |
| `name` | string | Internal camelCase identifier |
| `icon` | string or object | `'file:<name>.svg'` or `{ light: 'file:a.svg', dark: 'file:a.dark.svg' }` |
| `iconColor` | string | Preset theme color, e.g. `'orange-red'` (use instead of deprecated `defaults.color`) |
| `group` | string[] | `['transform']` for action nodes, `['trigger']` for triggers |
| `version` | number or number[] | Start at `1`; use array for light versioning |
| `subtitle` | string | Template shown below node name, e.g. `'={{$parameter["operation"]}}'` — required by lint |
| `description` | string | Short description for the node panel |
| `defaults` | object | `{ name: 'Display Name' }` |
| `inputs` | array | `[NodeConnectionTypes.Main]` |
| `outputs` | array | `[NodeConnectionTypes.Main]` |
| `usableAsTool` | boolean | Must be declared (lint error otherwise); set `true` unless there's a reason not to |
| `credentials` | array | `[{ name: 'credName', required: true }]` |
| `properties` | array | Resource, operation, and field definitions |

**For declarative nodes**, also add `requestDefaults` (baseURL, headers — supports dynamic expressions from credentials) and `routing` keys on operations and parameters. The full pattern language — `preSend`/`postReceive` functions, pagination, dynamic property paths with `$parent`/`$index`, declarative dropdowns, `resourceLocator`/`resourceMapper`/`fixedCollection`, and the `methods` object for `listSearch`/`loadOptions`/`resourceMapping` — is in `references/declarative-node.md`.

**For programmatic nodes**, also add an `async execute()` method with proper item looping via `this.getInputData()` and `pairedItem` linking — see `references/programmatic-node.md`.

### 3B: The Resource → Operation Pattern

n8n nodes follow a consistent UI pattern: **Resource** (what entity) → **Operation** (what action).

Each resource gets a dropdown, each operation gets a dropdown filtered by the selected resource using `displayOptions.show`. Operations should map to CRUD verbs where applicable: Create, Create or Update (Upsert), Delete, Get, Get Many, Update. Use the `action` field on each operation option to provide a human-readable description (e.g., `action: 'Create a contact'`). For Upsert, use displayName "Create or Update" with description "Create a new record or update an existing one (upsert)".

**Important naming rule:** The linter enforces naming list operations **"Get Many"** (not "Get All"). The operation value should be `getAll` but the display name must be `Get Many`.

### Return All / Limit Pattern

For list ("Get Many") operations, always include a `returnAll` boolean toggle (default `false`, description `'Whether to return all results or only up to a given limit'`) paired with a conditional `limit` number field that only shows when `returnAll` is `false` (`displayOptions: { show: { returnAll: [false] } }`). This is the standard pattern used across all n8n built-in nodes. See both reference templates for complete examples.

### 3C: displayOptions and Conditional Fields

Use `displayOptions.show` to conditionally display fields based on the selected resource, operation, or other parameter values (e.g., `show: { resource: ['contact'], operation: ['create'] }`). For version-specific fields, use `'@version'`: `displayOptions: { show: { '@version': [2] } }`. Advanced `_cnd` condition operators (gte, startsWith, regex, ...) and the `@tool`/`@feature` special keys are documented in `references/declarative-node.md`.

### 3D: Additional Fields (Optional Parameters)

Group optional parameters under a collection named "Additional Fields":

```typescript
{
  displayName: 'Additional Fields',
  name: 'additionalFields',
  type: 'collection',
  placeholder: 'Add Field',
  default: {},
  displayOptions: {
    show: { resource: ['contact'], operation: ['create'] },
  },
  options: [
    // Individual optional fields here
  ],
}
```

### 3E: The Codex File (`<Name>.node.json`)

Metadata controlling how the node appears in n8n's node discovery panel:

```json
{
  "node": "n8n-nodes-<package>",
  "nodeVersion": "1.0",
  "codexVersion": "1.0",
  "categories": ["Miscellaneous"],
  "subcategories": { "Miscellaneous": ["Helpers"] },
  "alias": ["synonym", "former-product-name"],
  "resources": {
    "credentialDocumentation": [{ "url": "" }],
    "primaryDocumentation": [{ "url": "" }]
  }
}
```

Set the `node` field to the npm package name (e.g., `"n8n-nodes-acme"`), as the official CLI templates do. At runtime n8n only consumes `categories`, `subcategories`, `resources`, and `alias` — `alias` strings are matched by the nodes-panel search, so add common synonyms and former product names.

Categories: Analytics, Communication, Data & Storage, Development, Finance & Accounting, Marketing & Content, Miscellaneous, Productivity, Sales, Utility.

### 3F: Credentials

Read `references/credentials.md` for complete patterns. Key points:
- File: `credentials/<Name>Api.credentials.ts`
- Class implements `ICredentialType`
- `name` must match the node's `credentials[].name` — and lint enforces naming: `name` starts lowercase and ends in `Api`, the class name ends in `Api` (OAuth2: `OAuth2Api`)
- Use `authenticate: IAuthenticateGeneric` for header/body/query auth
- Use `test: ICredentialTestRequest` to validate credentials — it runs automatically when the user saves the credential. Customize pass/fail with `test.rules`; use `testedBy` in the node for complex validation. OAuth2 credentials extending `oAuth2Api` are exempt (don't add `test` to them)
- For session-token APIs, use `preAuthentication` to exchange long-lived credentials for a token before requests
- Always use `$credentials` (plural) in expressions — `$credential` (singular) is wrong
- The linter requires an `icon` property using `Icon` type from n8n-workflow

### 3G: The Icon

Use a **square SVG** — the `icon-validation` lint rule errors on non-`.svg` icons (the docs' 60×60 PNG alternative fails lint in CLI-scaffolded projects and the verification scan). Place it alongside the `.node.ts` file and reference it with `icon: 'file:<name>.svg'`. For light/dark variants use `icon: { light: 'file:icon.svg', dark: 'file:icon.dark.svg' }` — the two must be different files. Set a brand-ish accent with the top-level `iconColor: '<preset>'` (e.g. `'orange-red'`, one of n8n's preset theme colors); `defaults.color` is deprecated. Don't reference Font Awesome — download and embed.

### 3H: Error Handling (Programmatic Nodes)

Use `NodeApiError` for API errors and `NodeOperationError` for validation errors (both from `n8n-workflow`). Inside per-item loops, always pass the item index: `throw new NodeApiError(this.getNode(), error as JsonObject, { itemIndex: i })` — the `node-operation-error-itemindex` lint rule requires it. Wrap each item's processing in `try/catch` and support `continueOnFail()` so users can choose to keep going on errors — push `{ json: { error: message }, pairedItem: { item: i } }` on failure. See `references/programmatic-node.md` → "Error Handling Patterns" for full examples including HTTP status-specific handling.

### 3I: Item Linking (pairedItem / constructExecutionMetaData)

Every output item in a programmatic node must link back to its source input — the `missing-paired-item` lint rule errors otherwise. Two approaches:

**Modern approach (recommended):** Use `constructExecutionMetaData`:
```typescript
const executionData = this.helpers.constructExecutionMetaData(
  this.helpers.returnJsonArray(responseData),
  { itemData: { item: i } },
);
returnData.push(...executionData);
```

**Manual approach:** Set `pairedItem` directly:
```typescript
returnData.push({
  json: responseData,
  pairedItem: { item: i },
});
```

For multi-input nodes add the input index: `pairedItem: { item: i, input: 0 }`. Without item linking, n8n can't trace data flow between nodes.

### 3J: HTTP Helpers

Use n8n's built-in helpers — no external HTTP libraries:

```typescript
// Without auth:
const response = await this.helpers.httpRequest(options);

// With auth (handles credential injection automatically):
const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'credentialTypeName', options
);
```

**Deprecation warning:** `this.helpers.requestWithAuthentication` and `IRequestOptions` are **deprecated** (the `no-deprecated-workflow-functions` lint rule flags them). Always use `httpRequestWithAuthentication` with `IHttpRequestOptions`. The new interface uses `url` (not `uri`) and defaults to JSON parsing.

For programmatic nodes, create a `GenericFunctions.ts` helper to centralize HTTP logic. Include `IHookFunctions`, `IWebhookFunctions`, and `IPollFunctions` in the `this` type union for trigger node compatibility. Use `loadOptionsMethod` for dropdowns that fetch values from an API at runtime. See `references/programmatic-node.md` for the complete patterns.

### 3K: Node Versioning

**Light versioning** (all node types): Change `version` to an array `[1, 2]` and use `displayOptions: { show: { '@version': [2] } }`.

**Full versioning** (programmatic only): Extend `VersionedNodeType` — imported from `n8n-workflow` — with separate `v1/`, `v2/` directories and a base description carrying `defaultVersion`. See the Mattermost node on GitHub for a real example and `references/programmatic-node.md` for the template.

**Feature-based versioning**: gate individual behaviors behind feature flags instead of whole versions — see `references/programmatic-node.md`.

### 3L: Hints and Notices

Four mechanisms guide users inside the editor: description-level `hints` (banners in input/output panes or the NDV, with `displayCondition` expressions), parameter-level `hint` text under a field, `type: 'notice'` inline info boxes between parameters, and dynamic post-execution hints via `this.addExecutionHints({ message, location })` inside `execute()` (programmatic only). See `references/declarative-node.md` → "Hints and Notices" and `references/programmatic-node.md`.

## Step 4: Validate (Gate Protocol)

After implementing, **prove** the node works. Run the gates in order; don't proceed past a failure. Full protocol with pass criteria and failure triage: `references/validation.md`.

```bash
npm run lint:fix && npm run lint                 # Gate 1 — strict-config guard + @n8n/eslint-plugin-community-nodes; exit 0, zero errors
npm run build                                    # Gate 2 — strict tsc + asset copy; '✓ Build successful' + dist paths exist
npx n8n-node cloud-support                       # Gate 3 — 'Cloud support is ENABLED' (strict mode + default eslint config)
npm run dev                                      # Gate 4 — local n8n at http://localhost:5678; smoke test node + credentials + operations
```

| Gate | Pass signal |
|------|-------------|
| 0. Structure sanity | package.json checklist (name, keywords, n8n block, peerDeps) all true |
| 1. Lint | Exit 0, zero errors — never edit `eslint.config.mjs`, never use inline `eslint-disable` (the verification scanner ignores them) |
| 2. Build | `✓ Build successful`; every `n8n.nodes`/`n8n.credentials` path exists in `dist/` |
| 3. Cloud support | `✅ Cloud support is ENABLED` |
| 4. Runtime | `Editor is now accessible`; node findable by displayName; credential test passes on save; operations execute |

**There is no unit-test command** in the n8n-node toolchain — CI runs exactly lint + build, and runtime validation is Gate 4. Don't invent a jest/vitest harness.

## Step 5: Release and Verify

```bash
npm run release                                  # Gate 5 — release-it: lint+build gates, version bump, changelog, tag, push, GitHub release
npx @n8n/scan-community-package <package-name>   # Gate 6 — post-publish: provenance check + re-lint of the published artifact
```

`npm run release` does **NOT** publish to npm from your machine — pushing the version tag triggers the scaffolded `.github/workflows/publish.yml`, which publishes with **npm provenance** (mandatory for verified community nodes since May 1, 2026). Plain `npm publish` is blocked by design via `prepublishOnly: n8n-node prerelease` — don't remove it. One-time setup: `npx n8n-node release --init-workflow` if `publish.yml` is missing, plus npm Trusted Publishing (or an `NPM_TOKEN` secret).

Read `references/publishing.md` for the full release flow, verification scope rules (one service per package, no logic nodes, npm↔GitHub identity), and the Creator Portal submission checklist.

## Code Standards Summary

- Write in TypeScript; use `import type` for type-only imports (a convention from n8n's codebase — keeps value vs type imports honest)
- Use `httpRequestWithAuthentication` (not the deprecated `requestWithAuthentication`); use `url` not `uri` in `IHttpRequestOptions`
- Never mutate incoming data — clone with spread or `structuredClone()`
- No runtime dependencies. For cloud-eligible nodes the lint allowlist is exactly: `n8n-workflow`, `@n8n/ai-node-sdk`, `lodash`, `moment`, `p-limit`, `luxon`, `zod`, `crypto`/`node:crypto`, plus relative imports — nothing else (no axios, no form-data; use built-in helpers and globals)
- No restricted globals in cloud-eligible nodes: use `sleep` from `n8n-workflow` instead of `setTimeout`; no `process`/`__dirname`; no `console.log` (lint error)
- `n8n-workflow` is a peer dependency, not bundled
- Follow Resource → Operation pattern with `noDataExpression: true` on selectors; include `action` on every operation option
- Name list operations **"Get Many"** (not "Get All") and use the `returnAll`/`limit` pair
- Use `NodeConnectionTypes.Main` (plural const, value import) — never the string `'main'`
- Declare `usableAsTool` on every node (lint error otherwise); set `true` unless there's a reason not to
- Use `constructExecutionMetaData` with `itemData` for item linking; every output item needs `pairedItem`
- Implement `continueOnFail()` in every execute loop; throw `NodeApiError`/`NodeOperationError` with `{ itemIndex }` in loops — never raw `Error`
- The `execute()` method returns `[returnData]` — don't forget the outer array wrapper
- Create `GenericFunctions.ts` for shared API request helpers (type `method: IHttpRequestMethods`; include `IHookFunctions`/`IWebhookFunctions`/`IPollFunctions` in the `this` union)
- Use `displayOptions` for progressive field disclosure; optional params go in "Additional Fields" collections
- Title Case for UI text; sentence case for descriptions/hints; placeholders start with `e.g. `
- Use `$credentials` (plural) in credential expressions; dynamic routing expressions need the `=` prefix
- Use `encodeURIComponent()` for user-provided values in routing URLs
- Declarative: use routing OR `execute()`, never both (defining `execute()` silently disables routing); `customOperations` is the per-operation escape hatch
- Split multi-resource nodes into `*Description.ts` files per resource, spread into the properties array
- Reuse internal parameter `value` names across operations
- Keep `"strict": true` in the `n8n` config of `package.json` and never edit `eslint.config.mjs`
- Pass the validation gates before publishing — see `references/validation.md` and `references/common-mistakes.md`

## UX Patterns (Verification Requirements)

These patterns are required for verified community nodes and recommended for all nodes:

**Delete operation output:** Always return `{ deleted: true }` (not `{ success: true }`) from Delete operations. This confirms the deletion and triggers the following node.

**Simplify toggle:** When an endpoint returns data with more than 10 fields, add a "Simplify" boolean parameter that returns a curated subset of max 10 fields. Use displayName `Simplify` and description `Whether to return a simplified version of the response instead of the raw data`. Flatten nested fields in simplified mode.

**AI Tool Output parameter:** For nodes used as AI agent tools, add an "Output" options parameter with three modes: Simplified (same as Simplify above), Raw (all fields), and Selected Fields (user picks which fields to send to the AI agent). This prevents context window overflow.

**Resource Locator:** Use `type: 'resourceLocator'` instead of a plain string input whenever a user needs to select a single item (e.g., a specific document, board, or channel). It offers ID, URL, and "From list" modes — default to "From list" when available. `methods.listSearch` is required when a list mode uses `searchListMethod`. See the Trello and Google Drive nodes for examples.

**Sorting options for Get Many:** Enhance list operations by providing sorting options in a dedicated collection below the main "Options" collection.

**Binary data naming:** Don't use "binary data" or "binary property" in field names. Instead use "Input Data Field Name" / "Output Data Field Name".

**Upsert:** When the API supports it, include "Create or Update" as a separate operation alongside Create and Update.

**Copy rules (the verification bar checks these):**
- Every `placeholder` starts with `e.g. ` — `placeholder: 'e.g. nathan@example.com'`
- Error messages state what happened AND how to fix it, quote the parameter's display name, and append `[Item X]` with the item index; never use the words "error", "problem", "failure", or "mistake" in messages or descriptions
- Order required fields by importance, then scope broad→narrow; sort optional fields alphabetically; document defaults in descriptions ("Defaults to false")
- Trigger nodes name the event parameter **"Trigger on"** (no tooltip)
- ID fields backed by a dynamic list are named `<Record> Name or ID` with description `Choose a name from the list, or specify an ID using an expression`
- Sort dropdown options alphabetically (also a lint warning); offer "All" instead of `*`
- Operation `action` phrasing omits articles and names the resource: `action: 'Update row in sheet'`

## Trigger Nodes

Triggers are always programmatic. Four patterns:

| Type | Method | Use When | Example |
|------|--------|----------|---------|
| Webhook (auto) | `webhook()` + `webhookMethods` | Service supports API-based webhook registration | Stripe Trigger |
| Webhook (manual) | `webhook()` only | User pastes webhook URL into external service | Generic Webhook |
| Polling | `poll()` | No webhook support; check for new data on a schedule | Gmail Trigger |
| Event/Stream | `trigger()` | Long-running connection (WebSocket, SSE, message queue) | AMQP Trigger |

**Key differences from action nodes:**
- Set `group: ['trigger']` and suffix the `displayName` with "Trigger"
- Trigger nodes have `inputs: []` — they have NO inputs
- Class names and filenames get the `Trigger` suffix (e.g., `MyServiceTrigger`)
- Use `getWorkflowStaticData('node')` to persist state (webhook IDs, last-checked timestamps) between calls
- Lint still requires `icon`, `subtitle`, and a declared `usableAsTool` (set `false` for triggers) on the description
- Webhook triggers must implement the complete `webhookMethods.default` lifecycle (`checkExists`/`create`/`delete`) — the `webhook-lifecycle-complete` lint rule errors otherwise, including on manual-URL webhook nodes

For complete trigger templates with full code examples, read `references/programmatic-node.md` → "Trigger Node Patterns".

## AI Sub-Nodes

To make a model or memory provider pluggable into the AI Agent: implement `supplyData(this: ISupplyDataFunctions, itemIndex)` instead of `execute()`, output an AI connection type (`outputs: [NodeConnectionTypes.AiLanguageModel]` or `AiMemory`), and declare `"aiNodeSdkVersion": 1` plus the `@n8n/ai-node-sdk` peer dependency in package.json (the `ai-node-package-json` lint rule enforces the pairing). Four CLI templates scaffold these. Read `references/ai-nodes.md`.

## Modular Structure (Complex Nodes)

For many resources/operations, split into modules:

```
nodes/MyNode/
├── MyNode.node.ts           # Main entry
├── GenericFunctions.ts      # Shared API request helpers
├── actions/                 # One dir per resource
│   ├── contact/
│   │   ├── create.ts
│   │   ├── get.ts
│   │   └── index.ts
│   └── deal/
│       └── index.ts
├── methods/                 # loadOptions, etc.
└── transport/               # Shared HTTP helpers
```

## n8n Data Structure

Data flows between nodes as arrays of items. Each item has `json` (required) and optionally `binary`. The `execute()` method returns `Promise<INodeExecutionData[][]>` — an array of arrays (one per output). Use `this.helpers.returnJsonArray(responseData)` to wrap raw data, and remember to return `[returnData]` (nested array).
