# Common Mistakes Reference

Error catalog for n8n node development. Each entry shows the wrong pattern and the correct fix.

For the full `@n8n/eslint-plugin-community-nodes` rule catalog and the gate-by-gate validation protocol (lint, build, dev runtime, cloud-support, release prechecks, verification scan), see `references/validation.md`. AI sub-node specifics live in `references/ai-nodes.md`.

## Table of Contents

1. [File Structure Errors](#file-structure-errors)
2. [Description Errors](#description-errors)
3. [Execute Method Errors](#execute-method-errors)
4. [Credential Errors](#credential-errors)
5. [Declarative Node Errors](#declarative-node-errors)
6. [Linter Errors](#linter-errors)
7. [Publishing Errors](#publishing-errors)
8. [Quick Fix Reference](#quick-fix-reference)

## File Structure Errors

### 1. Class name doesn't match filename

**Wrong:**
```
File: MyService.node.ts
Class: export class MyServiceNode implements INodeType  // "Node" suffix doesn't match
```

**Fix:** Class name must exactly match the filename (minus `.node.ts`):
```
File: MyService.node.ts
Class: export class MyService implements INodeType
```

### 2. Wrong npm package prefix

**Wrong:** `"name": "myservice-n8n-nodes"` or `"name": "n8n-myservice"`

**Fix:** Package name must be `n8n-nodes-<name>` or the scoped form `@<org>/n8n-nodes-<name>` — both the CLI name validation and the `package-name-convention` lint rule accept either:
```json
"name": "n8n-nodes-myservice"
```
```json
"name": "@mycompany/n8n-nodes-myservice"
```

### 3. Missing codex file

Every node should ship a `.node.json` codex file alongside the `.node.ts` file (n8n recommends it). Without it the node still loads and still appears in the nodes panel (search matches the node name), but it won't be categorized, has no search aliases (codex `alias`), and lacks credential/primary documentation links.

### 4. Wrong paths in package.json n8n config

**Wrong:** Pointing to source files:
```json
"nodes": ["nodes/MyService/MyService.node.ts"]
```

**Fix:** Point to compiled output:
```json
"nodes": ["dist/nodes/MyService/MyService.node.js"]
```

## Description Errors

### 5. Missing noDataExpression on selectors

**Wrong:**
```typescript
{ displayName: 'Resource', name: 'resource', type: 'options', /* ... */ }
```

**Fix:** Always set `noDataExpression: true` on resource and operation selectors:
```typescript
{ displayName: 'Resource', name: 'resource', type: 'options', noDataExpression: true, /* ... */ }
```

### 6. Missing action field on operations

**Wrong:**
```typescript
options: [
  { name: 'Create', value: 'create' },
]
```

**Fix:** Every operation option needs an `action` field:
```typescript
options: [
  { name: 'Create', value: 'create', action: 'Create a contact' },
]
```

### 7. NodeConnectionType used as a value (it's type-only)

In current `n8n-workflow`, `NodeConnectionType` is exported **only as a type** — using it as a value fails to compile: `'NodeConnectionType' cannot be used as a value because it was exported using 'export type'`. The runtime value export is the plural const object `NodeConnectionTypes`. (Only very old 1.x versions exported a usable `NodeConnectionType` enum.)

**Wrong:**
```typescript
import { NodeConnectionType } from 'n8n-workflow';

inputs: [NodeConnectionType.Main],   // Compile error: type-only export
```

**Also wrong — string literals trigger the lint error `@n8n/community-nodes/node-connection-type-literal`:**
```typescript
inputs: ['main'],
outputs: ['main'],
```

**Fix:** Import the plural const and use it everywhere:
```typescript
import { NodeConnectionTypes } from 'n8n-workflow';

inputs: [NodeConnectionTypes.Main],
outputs: [NodeConnectionTypes.Main],
```

The `node-connection-type-literal` lint rule is autofixable (`npm run lint:fix`).

### 8. Trigger node with non-empty inputs

**Wrong:**
```typescript
// Trigger node:
inputs: [NodeConnectionTypes.Main],  // Triggers don't have inputs
```

**Fix:**
```typescript
inputs: [],  // Trigger nodes have NO inputs
```

### 9. Expression prefix missing for dynamic URLs

**Wrong:**
```typescript
url: '/contacts/{{$parameter["contactId"]}}'  // Missing = prefix
```

**Fix:** Dynamic expressions in routing must start with `=`:
```typescript
url: '=/contacts/{{$parameter["contactId"]}}'  // = prefix required
```

### 43. Placeholder text not starting with "e.g."

n8n's UX guidelines (checked during verification review) require parameter `placeholder` values that show example content to start with `e.g.`:

**Wrong:**
```typescript
{ displayName: 'Email', name: 'email', type: 'string', default: '',
  placeholder: 'nathan@example.com' }
```

**Fix:**
```typescript
{ displayName: 'Email', name: 'email', type: 'string', default: '',
  placeholder: 'e.g. nathan@example.com' }
```

## Execute Method Errors

### 10. Missing continueOnFail handling

**Wrong:**
```typescript
for (let i = 0; i < items.length; i++) {
  const data = await apiRequest.call(this, 'GET', '/items');
  returnData.push(...data);  // No error handling, no item linking
}
```

**Fix:** Wrap each item in try/catch with `continueOnFail()`:
```typescript
for (let i = 0; i < items.length; i++) {
  try {
    const data = await apiRequest.call(this, 'GET', '/items');
    const executionData = this.helpers.constructExecutionMetaData(
      this.helpers.returnJsonArray(data),
      { itemData: { item: i } },
    );
    returnData.push(...executionData);
  } catch (error) {
    if (this.continueOnFail()) {
      returnData.push(...this.helpers.constructExecutionMetaData(
        this.helpers.returnJsonArray({ error: (error as Error).message }),
        { itemData: { item: i } },
      ));
      continue;
    }
    throw error;
  }
}
```

### 11. Missing constructExecutionMetaData

**Wrong:**
```typescript
returnData.push(...this.helpers.returnJsonArray(responseData));  // No item linking
```

**Fix:** Always wrap with `constructExecutionMetaData` for proper item tracking:
```typescript
const executionData = this.helpers.constructExecutionMetaData(
  this.helpers.returnJsonArray(responseData),
  { itemData: { item: i } },
);
returnData.push(...executionData);
```

### 12. Not returning nested array

**Wrong:**
```typescript
return returnData;  // Must be INodeExecutionData[][]
```

**Fix:** The `execute()` method must return an array of arrays (one per output connector):
```typescript
return [returnData];  // Wrap in outer array
```

### 13. Delete operation returning wrong output

**Wrong:**
```typescript
returnData.push({ json: { success: true }, pairedItem: { item: i } });
```

**Fix:** Delete operations must return `{ deleted: true }` per n8n UX guidelines:
```typescript
returnData.push({ json: { deleted: true }, pairedItem: { item: i } });
```

This confirms the deletion succeeded and ensures the next node in the workflow receives a trigger item.

## Credential Errors

### 14. Wrong credential expression syntax

**Wrong:**
```typescript
headers: { Authorization: '={{$credential.apiKey}}' }   // singular
```

**Fix:** Always use `$credentials` (plural):
```typescript
headers: { Authorization: '={{$credentials.apiKey}}' }   // plural: $credentials
```

### 15. Missing password typeOptions on secrets

**Wrong:**
```typescript
{ displayName: 'API Key', name: 'apiKey', type: 'string', default: '' }
```

**Fix:**
```typescript
{ displayName: 'API Key', name: 'apiKey', type: 'string',
  typeOptions: { password: true }, default: '' }
```

### 16. Credential not registered in package.json

Even if the credential file exists, it won't load unless listed:
```json
"n8n": {
  "credentials": ["dist/credentials/MyServiceApi.credentials.js"]
}
```

### 17. Missing icon on credential class

The linter requires credentials to have an `icon` property:

```typescript
import type { Icon } from 'n8n-workflow';

export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'My Service API';
  icon: Icon = 'file:myservice.svg';  // SVG must be in credentials/ folder
  // ...
}
```

## Declarative Node Errors

### 18. Including execute() in a declarative node

If a node defines `execute()`, n8n runs `execute()` and the declarative routing/`requestDefaults` config is silently **ignored** — n8n only auto-assigns the routing-based execution when the node has no `execute`/`supplyData`/`poll`/`trigger` method. Either use routing OR execute, not both; when both are present, the leftover routing config is the part that does nothing.

### 19. Missing routing on operation options

**Wrong (declarative):**
```typescript
options: [{ name: 'Create', value: 'create', action: 'Create item' }]  // No routing
```

**Fix:**
```typescript
options: [{
  name: 'Create', value: 'create', action: 'Create item',
  routing: { request: { method: 'POST', url: '/items' } },
}]
```

### 25. Wrong preSend function signature

**Wrong:**
```typescript
// Missing proper this type, wrong return type
async function myPreSend(requestOptions: IHttpRequestOptions) {
  requestOptions.body = { data: 'test' };
  // Forgot to return requestOptions
}
```

**Fix:** preSend functions must use `IExecuteSingleFunctions` as `this` and return `Promise<IHttpRequestOptions>`:
```typescript
export const myPreSend = async function (
  this: IExecuteSingleFunctions,
  requestOptions: IHttpRequestOptions,
): Promise<IHttpRequestOptions> {
  requestOptions.body = { data: 'test' };
  return requestOptions;  // Must return the modified options
};
```

### 26. returnFullResponse confusion (declarative vs programmatic)

The declarative routing engine forces `returnFullResponse: true` internally on **every** request, so custom postReceive functions ALWAYS receive the full `IN8nHttpFullResponse` (body, headers, statusCode) — setting `returnFullResponse` in `routing.request` is a no-op. The option only matters for direct `this.helpers.httpRequest` / `httpRequestWithAuthentication` calls in programmatic code.

**Wrong (programmatic node that needs response headers):**
```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(this, 'myServiceApi', {
  method: 'GET',
  url: 'https://api.example.com/files/download',
  encoding: 'arraybuffer',
});
// response is just the body — no headers or statusCode
```

**Fix:** Set `returnFullResponse: true` on the programmatic request:
```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(this, 'myServiceApi', {
  method: 'GET',
  url: 'https://api.example.com/files/download',
  returnFullResponse: true,  // response = { body, headers, statusCode }
  encoding: 'arraybuffer',
});
```

### 27. Reading resourceLocator value without extractValue

**Wrong:**
```typescript
// In a listSearch or preSend method:
const teamId = this.getCurrentNodeParameter('teamId') as string;
// Returns { mode: 'list', value: 'abc123' } — an object, not a string!
```

**Fix:** Use `.value` on the returned object, or use `{ extractValue: true }`:
```typescript
// Option A: Access .value directly
const teamIdParam = this.getCurrentNodeParameter('teamId') as INodeParameterResourceLocator;
const teamId = teamIdParam.value as string;

// Option B: Use extractValue option (when available)
const teamId = this.getCurrentNodeParameter('teamId', { extractValue: true }) as string;
```

### 28. Misunderstanding when declarative pagination triggers (it's opt-in)

Pagination is **opt-in**: it only runs when `requestData.paginate` resolves truthy AND `operations.pagination` is defined. `paginate` defaults to undefined (falsy), so a Create/Update/Delete operation with no `routing.send.paginate` anywhere will never paginate — even when `requestOperations.pagination` is configured node-wide. Don't sprinkle `paginate: false` everywhere "just in case".

**When `paginate: false` IS needed:** when another *displayed* property sets `paginate` truthy for this operation — e.g. a shared "Return All" field whose `displayOptions` also show it for this operation. The first `paginate` value set wins, so an explicit `false` on the operation option blocks the shared field's `true`:

```typescript
options: [{
  name: 'Create', value: 'create', action: 'Create a record',
  routing: {
    request: { method: 'POST', url: '/records' },
    send: {
      paginate: false,  // First value set wins — blocks a displayed shared "Return All" field
      preSend: [createRecordBody],
      type: 'body',
    },
  },
}]
```

Hidden properties are skipped entirely, so if the shared field isn't displayed for this operation, no defensive flag is needed.

### 29. Wrong postReceive function signature

**Wrong:**
```typescript
// Treating postReceive like preSend
async function handleResponse(
  this: IExecuteSingleFunctions,
  requestOptions: IHttpRequestOptions,
): Promise<IHttpRequestOptions> { ... }
```

**Fix:** Custom postReceive functions receive `(items, response)` and return `INodeExecutionData[]`:
```typescript
export const handleResponse = async function (
  this: IExecuteSingleFunctions,
  items: INodeExecutionData[],
  response: IN8nHttpFullResponse,
): Promise<INodeExecutionData[]> {
  // Transform items based on response
  return items;
};
```

### 30. Missing URL encoding for user-provided values in routing URLs

**Wrong:**
```typescript
url: '=/v3/contacts/{{$parameter.identifier}}'  // Email with @ will break the URL
```

**Fix:** Use `encodeURIComponent()` for user-provided values that may contain special characters:
```typescript
url: '=/v3/contacts/{{encodeURIComponent($parameter.identifier)}}'
```

### 31. Wrong property path expression for dynamic nested body fields

**Wrong:**
```typescript
routing: {
  send: {
    property: 'attributes.$parent.fieldName',  // Literal string, not evaluated
    type: 'body',
  },
}
```

**Fix:** Dynamic property paths must start with `=` and use `{{}}` for expressions:
```typescript
routing: {
  send: {
    property: '=attributes.{{$parent.fieldName}}',  // Evaluates to body.attributes[selectedField]
    type: 'body',
  },
}
```

Same for array indexing:
```typescript
property: '=items[{{$index}}].value',  // Not 'items[$index].value'
```

### 32. Missing ignoreHttpStatusErrors for custom error postReceive

**Wrong:**
```typescript
routing: {
  request: {
    method: 'DELETE',
    url: '=/items/{{$parameter.itemId}}',
    // Missing ignoreHttpStatusErrors — n8n throws before postReceive runs
  },
  output: {
    postReceive: [handleErrors],  // Never reached on 4xx/5xx
  },
}
```

**Fix:** Add `ignoreHttpStatusErrors: true` so your custom postReceive function can inspect the response:
```typescript
routing: {
  request: {
    method: 'DELETE',
    url: '=/items/{{$parameter.itemId}}',
    ignoreHttpStatusErrors: true,
  },
  output: {
    postReceive: [handleErrors],  // Now receives 4xx/5xx responses
  },
}
```

### 33. Offset pagination missing rootProperty

**Wrong:**
```typescript
operations: {
  pagination: {
    type: 'offset',
    properties: {
      limitParameter: 'limit',
      offsetParameter: 'offset',
      pageSize: 100,
      type: 'query',
      // Missing rootProperty — pagination can't find items in nested response
    },
  },
}
// API returns: { data: { items: [...] } }
```

**Fix:** Set `rootProperty` to the JSON path where the items array lives:
```typescript
operations: {
  pagination: {
    type: 'offset',
    properties: {
      limitParameter: 'limit',
      offsetParameter: 'offset',
      pageSize: 100,
      rootProperty: 'data.items',
      type: 'query',
    },
  },
}
```

### 34. Generic pagination continue expression always true

**Wrong:**
```typescript
operations: {
  pagination: {
    type: 'generic',
    properties: {
      continue: '={{ $response.body.nextPageToken }}',  // Non-empty string is truthy
      request: { qs: { pageToken: '={{ $response.body.nextPageToken }}' } },
    },
  },
}
```

**Fix:** Use `!!` to coerce to boolean, so empty strings and undefined become `false`:
```typescript
operations: {
  pagination: {
    type: 'generic',
    properties: {
      continue: '={{ !!$response.body?.nextPageToken }}',
      request: { qs: { pageToken: '={{ $response.body?.nextPageToken ?? "" }}' } },
    },
  },
}
```

### 35. Custom pagination function missing makeRoutingRequest

**Wrong:**
```typescript
// Using httpRequest directly in pagination — skips auth, preSend, postReceive
operations: {
  pagination: async function(this, requestOptions) {
    const response = await this.helpers.httpRequest(requestOptions.options);  // Wrong
    // ...
  },
}
```

**Fix:** Use `this.makeRoutingRequest()` which delegates to n8n's routing engine (handles auth, preSend, postReceive automatically):
```typescript
operations: {
  pagination: async function(this, requestOptions) {
    const responseData = await this.makeRoutingRequest(requestOptions);  // Correct
    // ...
  },
}
```

### 36. Using propertyInDotNotation when dots are literal

**Wrong:**
```typescript
// API field name contains a literal dot: "custom.field"
routing: {
  send: {
    property: 'custom.field',  // Creates body.custom.field (nested) instead of body["custom.field"]
    type: 'body',
  },
}
```

**Fix:** Set `propertyInDotNotation: false` when property names contain literal dots:
```typescript
routing: {
  send: {
    property: 'custom.field',
    propertyInDotNotation: false,  // Treats "custom.field" as a flat key
    type: 'body',
  },
}
```

## Linter Errors

### 20. Using deprecated request APIs

**Wrong:**
```typescript
import { IRequestOptions } from 'n8n-workflow';

const options: IRequestOptions = {
  method: 'GET',
  uri: 'https://api.example.com/items',
  json: true,
};
const response = await this.helpers.requestWithAuthentication.call(this, 'myServiceApi', options);
```

**Fix:** Use `IHttpRequestOptions` and `httpRequestWithAuthentication`:
```typescript
import type { IHttpRequestOptions } from 'n8n-workflow';

const options: IHttpRequestOptions = {
  method: 'GET',
  url: 'https://api.example.com/items',  // 'url' not 'uri'
  // No 'json: true' needed — JSON is the default
};
const response = await this.helpers.httpRequestWithAuthentication.call(this, 'myServiceApi', options);
```

### 21. Wrong list operation naming

**Wrong:**
```typescript
{ name: 'Get All', value: 'getAll', action: 'Get all contacts', description: 'Get all contacts' }
```

**Fix:** The linter enforces "Get Many":
```typescript
{ name: 'Get Many', value: 'getAll', action: 'Get many contacts', description: 'Get many contacts' }
```

### 22. Missing `import type` for type-only imports (convention, not enforced)

The current standard lint setup (the `eslint.config.mjs` supplied by `@n8n/node-cli`, with `@n8n/eslint-plugin-community-nodes` and the legacy n8n-nodes-base rule sets) does NOT enforce `import type` — the old n8n-nodes-starter `.eslintrc.js` did, via `@typescript-eslint/consistent-type-imports`. The n8n templates and built-in nodes still use `import type` consistently, so follow it as a convention:

**Inconsistent with templates:**
```typescript
import { INodeType, INodeTypeDescription, INodeExecutionData } from 'n8n-workflow';
// INodeExecutionData only used in type annotations, not at runtime
```

**Preferred:**
```typescript
import type { INodeExecutionData } from 'n8n-workflow';
import { INodeType, INodeTypeDescription } from 'n8n-workflow';
```

**Rule of thumb:** If a symbol is only used in `: TypeName` annotations, function signatures, or `as TypeName` casts, import it with `import type`. If it's used as a value (e.g., `throw new NodeOperationError(...)`, `NodeConnectionTypes.Main`), use a regular import.

### 23. `no-credential-reuse` false positive on Windows

The `no-credential-reuse` rule has a bug when the project sits at the first directory level under a Windows drive root (e.g., `D:\my-project`).

**Workaround:** Either move the project deeper (e.g., `D:\projects\my-project`) or add an eslint-disable block:
```typescript
/* eslint-disable @n8n/community-nodes/no-credential-reuse */
credentials: [
  { name: 'myServiceApi', required: true },
],
/* eslint-enable @n8n/community-nodes/no-credential-reuse */
```

Note: inline disables only help local lint — n8n's verification scanner ignores them (see mistake 38). That's fine here because this Windows-only false positive doesn't reproduce in the scanner's environment.

### 37. PNG icon fails `icon-validation`

The `@n8n/community-nodes/icon-validation` lint rule requires icons to be SVG files that exist on disk — `.png` (or any non-`.svg`) fails with `Icon file "..." must be an SVG file (end with .svg)`. With the light/dark object form, the two paths must be **different files** (`Light and dark icons cannot be the same file`).

**Wrong:**
```typescript
icon: 'file:myservice.png',
// or:
icon: { light: 'file:myservice.svg', dark: 'file:myservice.svg' },  // same file
```

**Fix:**
```typescript
icon: 'file:myservice.svg',
// or, with a real dark variant:
icon: { light: 'file:myservice.svg', dark: 'file:myservice.dark.svg' },
```

### 38. Importing `form-data` (or any non-allowlisted package)

The cloud-only rule `@n8n/community-nodes/no-restricted-imports` rejects every import/require/dynamic import that isn't on the allowlist (`n8n-workflow`, `@n8n/ai-node-sdk`, `lodash`, `moment`, `p-limit`, `luxon`, `zod`, `crypto`/`node:crypto`, and relative paths): "n8n Cloud does not allow community nodes with dependencies." `form-data` is the classic offender.

Do NOT reach for `// eslint-disable-next-line` — n8n's verification scanner (`@n8n/scan-community-package`) runs ESLint with `allowInlineConfig: false`, so inline disable comments are **ignored** and the scan still fails.

**Wrong:**
```typescript
import FormData from 'form-data';
```

**Fix:** Use the Node 18+ globals — no import needed:
```typescript
const form = new FormData();
form.append('file', new Blob([buffer]), 'report.pdf');
```

### 39. NodeApiError/NodeOperationError in item loops without `{ itemIndex }`

The `@n8n/community-nodes/node-operation-error-itemindex` rule (error) flags `new NodeOperationError(...)` or `new NodeApiError(...)` inside item loops in `execute()` that omit `{ itemIndex }` as the third argument. Without it, n8n can't associate the error with the failing item, breaking per-item error reporting and `continueOnFail`.

**Wrong:**
```typescript
for (let i = 0; i < items.length; i++) {
  throw new NodeOperationError(this.getNode(), 'Invalid input');
}
```

**Fix:**
```typescript
for (let i = 0; i < items.length; i++) {
  throw new NodeOperationError(this.getNode(), 'Invalid input', { itemIndex: i });
}
```

### 40. Webhook trigger with incomplete `webhookMethods`

The `@n8n/community-nodes/webhook-lifecycle-complete` rule (error) requires webhook-based trigger nodes (description declares a non-empty `webhooks` array) to implement the full `webhookMethods` lifecycle — `checkExists`, `create`, AND `delete` in each group. Polling triggers (no `webhooks` array) are exempt.

**Fix:** Implement all three methods:
```typescript
webhookMethods = {
  default: {
    async checkExists(this: IHookFunctions): Promise<boolean> { /* verify webhook on remote service */ },
    async create(this: IHookFunctions): Promise<boolean> { /* register webhook */ },
    async delete(this: IHookFunctions): Promise<boolean> { /* clean up webhook */ },
  },
};
```

### 41. Editing `eslint.config.mjs` → "Strict mode violation"

With `"strict": true` under the `n8n` section of package.json (the scaffold default), `npm run lint` first compares `eslint.config.mjs` against the default template and exits 1 with `Strict mode violation: eslint.config.mjs has been modified from the default configuration` — **before ESLint even runs**. Never edit `eslint.config.mjs`.

**Fix:** Restore the default config:
```bash
npx n8n-node cloud-support enable
```

To intentionally opt out of the cloud-only rules, use `npx n8n-node cloud-support disable` (switches config and strict mode for you) — don't hand-edit the config.

### 42. Using setTimeout/setInterval

The cloud-only rule `@n8n/community-nodes/no-restricted-globals` forbids `setTimeout`, `setInterval`, `clearTimeout`, `clearInterval`, `setImmediate`, `clearImmediate`, `process`, `global`/`globalThis`, `__dirname`, and `__filename`.

**Wrong:**
```typescript
await new Promise((resolve) => setTimeout(resolve, 1000));
```

**Fix:** Use `sleep` from `n8n-workflow`:
```typescript
import { sleep } from 'n8n-workflow';

await sleep(1000);
```

### 44. Trigger node description missing icon/subtitle/usableAsTool

Easy to forget on trigger nodes, but these rules apply to ALL node classes:

- `@n8n/community-nodes/require-node-description-fields` (error): the description must define `icon` and `subtitle`.
- `@n8n/community-nodes/node-usable-as-tool` (error, autofixable): the description must set `usableAsTool`. Only AI sub-nodes (empty inputs + non-Main outputs) are exempt — a regular trigger with `outputs: [NodeConnectionTypes.Main]` is not.

**Fix:**
```typescript
description: INodeTypeDescription = {
  displayName: 'My Service Trigger',
  name: 'myServiceTrigger',
  icon: 'file:myservice.svg',
  subtitle: '={{$parameter["event"]}}',
  usableAsTool: true,
  group: ['trigger'],
  inputs: [],
  outputs: [NodeConnectionTypes.Main],
  // ...
};
```

## Publishing Errors

### 24. `prepublishOnly` script blocks `npm publish`

Projects scaffolded with `npm create @n8n/node` (the n8n-node CLI templates) include a `prepublishOnly` script that runs `n8n-node prerelease`, which blocks direct `npm publish`. That's by design.

Locally, `npm run release` (= `n8n-node release`) does **NOT** publish to npm: release-it runs with `--npm.publish=false` and only bumps the version, generates the changelog, commits, tags, pushes, and creates a GitHub release. The tag then triggers the scaffolded `.github/workflows/publish.yml` (scaffold it with `n8n-node release --init-workflow` if missing), which publishes to npm with provenance — `RELEASE_MODE=true` is set there so `prerelease` passes.

**Fix:** Run `npm run release` locally and let GitHub Actions do the npm publish.

Do NOT remove `prepublishOnly` to run `npm publish --access public` directly: that skips the lint/build prechecks and produces a package WITHOUT npm provenance, which fails n8n's verification scanner — and from May 1, 2026, all community nodes must be published via GitHub Actions with npm provenance. `n8n-node release --publish` exists only as a last-resort escape hatch for private/unverified packages (no provenance; explicitly discouraged).

See `references/publishing.md` for the full release flow and `references/validation.md` for the release-readiness gate.

## Quick Fix Reference

| Error | Fix |
|-------|-----|
| Node not appearing in editor | Check `package.json` n8n.nodes paths, rebuild, restart |
| "Cannot find credential" | Check credential `name` matches between node and credential file |
| Empty response data | Check `postReceive` rootProperty extracts correct JSON path |
| Item linking broken | Add `constructExecutionMetaData` with `{ itemData: { item: i } }` |
| displayOptions not working | Verify resource/operation values match exactly (case-sensitive) |
| Expression not resolving | Use `=` prefix: `'=/path/{{$parameter.id}}'` not `'/path/{{$parameter.id}}'` |
| `requestWithAuthentication` deprecated | Switch to `httpRequestWithAuthentication` with `IHttpRequestOptions` |
| `uri` property error | Use `url` instead of `uri` in `IHttpRequestOptions` |
| `node-usable-as-tool` lint error | Add `usableAsTool: true` to node description (autofixable) |
| "Get All" lint error | Change to "Get Many" / "Get many" in name, action, description |
| `no-credential-reuse` false positive | Move project deeper than drive root, or eslint-disable (local lint only) |
| `prepublishOnly` blocks publish | Keep the script; use `npm run release` — GitHub Actions publishes to npm with provenance |
| `NodeConnectionType` cannot be used as a value | Import the plural const `NodeConnectionTypes` from `n8n-workflow` — never string `'main'` (lint error `node-connection-type-literal`) |
| `execute()` + `requestDefaults` both present | `execute()` wins, routing is silently ignored — remove one (declarative nodes must not define `execute()`) |
| `$credential` not resolving | Use `$credentials` (plural) in expressions |
| Return type error in execute | Return `[returnData]` not `returnData` |
| Delete returns `{ success: true }` | Use `{ deleted: true }` per n8n UX guidelines |
| preSend not modifying request | Return the modified `requestOptions` — forgetting to return is a common error |
| Need response headers in programmatic code | Add `returnFullResponse: true` to the `httpRequest` options (declarative postReceive always gets the full response) |
| resourceLocator returns object | Use `.value` on the result or pass `{ extractValue: true }` to `getCurrentNodeParameter` |
| Create/Update triggering pagination | Only happens when a displayed property sets `paginate` truthy — add `paginate: false` on the operation (first value set wins) |
| Custom postReceive wrong signature | Use `(items: INodeExecutionData[], response: IN8nHttpFullResponse)` not `(requestOptions)` |
| URL breaks with special characters | Use `encodeURIComponent()` in routing URL expressions for user values |
| Dynamic property path not evaluated | Use `=` prefix and `{{}}`: `'=attributes.{{$parent.fieldName}}'` not `'attributes.$parent.fieldName'` |
| Custom error handler never reached | Add `ignoreHttpStatusErrors: true` to the operation's `routing.request` |
| Offset pagination returns no data | Set `rootProperty` to the JSON path of the items array in the response |
| Generic pagination loops forever | Use `!!` in `continue` expression: `'={{ !!$response.body?.nextToken }}'` |
| Custom pagination skips auth | Use `this.makeRoutingRequest()` not `this.helpers.httpRequest()` |
| Dot in property name creates nesting | Set `propertyInDotNotation: false` on `routing.send` for literal dots |
| `icon-validation` lint error | Use SVG icons (no PNG); light and dark variants must be different files |
| `no-restricted-imports` on `form-data` | Use global `FormData`/`Blob` — the verification scanner ignores inline `eslint-disable` comments |
| `node-operation-error-itemindex` lint error | Pass `{ itemIndex: i }` as third argument to `NodeApiError`/`NodeOperationError` in item loops |
| `webhook-lifecycle-complete` lint error | Implement all of `checkExists`, `create`, `delete` in `webhookMethods` |
| "Strict mode violation" exit 1 before ESLint runs | Never edit `eslint.config.mjs` — restore with `npx n8n-node cloud-support enable` |
| `setTimeout` flagged by `no-restricted-globals` | Use `sleep` from `n8n-workflow` |
| Placeholder rejected in verification review | Start placeholders with `e.g.` per UX guidelines |
| `require-node-description-fields` lint error | Add `icon` and `subtitle` to the node description (trigger nodes too) |

Full lint rule catalog and validation gate protocol: `references/validation.md`.
