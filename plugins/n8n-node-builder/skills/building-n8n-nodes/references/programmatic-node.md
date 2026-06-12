# Programmatic Node Reference

Complete template and patterns for building a programmatic-style n8n node.

## Table of Contents

1. [Complete Base File Template](#complete-base-file-template)
2. [The execute() Method](#the-execute-method)
3. [Item Processing Loop](#item-processing-loop)
4. [Error Handling Patterns](#error-handling-patterns)
5. [HTTP Request Patterns](#http-request-patterns)
6. [Item Linking (constructExecutionMetaData)](#item-linking-constructexecutionmetadata)
7. [Binary Data Handling](#binary-data-handling)
8. [Trigger Node Patterns](#trigger-node-patterns)
9. [Full Versioning Structure](#full-versioning-structure)
10. [GenericFunctions.ts Pattern](#genericfunctionsts-pattern)
11. [Complete `this.*` Methods Reference](#complete-this-methods-reference)
12. [Complete `this.helpers` Reference](#complete-thishelpers-reference)
13. [Dynamic Options (loadOptions & listSearch)](#dynamic-options-loadoptions--listsearch)
14. [Multiple Outputs](#multiple-outputs)
15. [Execution Control](#execution-control)
16. [Advanced Binary Data Patterns](#advanced-binary-data-patterns)
17. [Advanced GenericFunctions Patterns](#advanced-genericfunctions-patterns)

## Complete Base File Template

```typescript
import type { IExecuteFunctions } from 'n8n-workflow';
import {
  IDataObject,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
  JsonObject,
  NodeApiError,
  NodeConnectionTypes,
  NodeOperationError,
} from 'n8n-workflow';

export class MyService implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Service',
    name: 'myService',
    icon: 'file:myService.svg',
    group: ['transform'],
    version: 1,
    subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
    description: 'Interact with the My Service API',
    defaults: {
      name: 'My Service',
    },
    inputs: [NodeConnectionTypes.Main],
    outputs: [NodeConnectionTypes.Main],
    usableAsTool: true,
    credentials: [
      {
        name: 'myServiceApi',
        required: true,
      },
    ],
    properties: [
      // Resource
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        noDataExpression: true,
        options: [
          {
            name: 'Contact',
            value: 'contact',
          },
        ],
        default: 'contact',
      },
      // Operations
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: {
          show: {
            resource: ['contact'],
          },
        },
        options: [
          {
            name: 'Create',
            value: 'create',
            action: 'Create a contact',
            description: 'Create a new contact',
          },
          {
            name: 'Get',
            value: 'get',
            action: 'Get a contact',
            description: 'Retrieve a contact by ID',
          },
          {
            name: 'Get Many',
            value: 'getAll',
            action: 'Get many contacts',
            description: 'Retrieve multiple contacts',
          },
          {
            name: 'Update',
            value: 'update',
            action: 'Update a contact',
            description: 'Update an existing contact',
          },
          {
            name: 'Delete',
            value: 'delete',
            action: 'Delete a contact',
            description: 'Delete a contact',
          },
        ],
        default: 'create',
      },
      // Fields
      {
        displayName: 'Contact ID',
        name: 'contactId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: {
          show: {
            resource: ['contact'],
            operation: ['get', 'update', 'delete'],
          },
        },
      },
      {
        displayName: 'Email',
        name: 'email',
        type: 'string',
        required: true,
        default: '',
        placeholder: 'e.g. user@example.com',
        displayOptions: {
          show: {
            resource: ['contact'],
            operation: ['create'],
          },
        },
      },
      {
        displayName: 'Additional Fields',
        name: 'additionalFields',
        type: 'collection',
        placeholder: 'Add Field',
        default: {},
        displayOptions: {
          show: {
            resource: ['contact'],
            operation: ['create', 'update'],
          },
        },
        options: [
          {
            displayName: 'First Name',
            name: 'firstName',
            type: 'string',
            default: '',
          },
          {
            displayName: 'Last Name',
            name: 'lastName',
            type: 'string',
            default: '',
          },
        ],
      },
      {
        displayName: 'Return All',
        name: 'returnAll',
        type: 'boolean',
        default: false,
        displayOptions: {
          show: { resource: ['contact'], operation: ['getAll'] },
        },
        description: 'Whether to return all results or only up to a given limit',
      },
      {
        displayName: 'Limit',
        name: 'limit',
        type: 'number',
        typeOptions: { minValue: 1 },
        default: 50,
        displayOptions: {
          show: {
            resource: ['contact'],
            operation: ['getAll'],
            returnAll: [false],
          },
        },
        description: 'Max number of results to return',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    const resource = this.getNodeParameter('resource', 0) as string;
    const operation = this.getNodeParameter('operation', 0) as string;

    for (let i = 0; i < items.length; i++) {
      try {
        if (resource === 'contact') {
          if (operation === 'create') {
            const email = this.getNodeParameter('email', i) as string;
            const additionalFields = this.getNodeParameter(
              'additionalFields', i,
            ) as IDataObject;

            const body: IDataObject = { email, ...additionalFields };

            const response = await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myServiceApi',
              {
                method: 'POST',
                url: 'https://api.myservice.com/v1/contacts',
                body,
              },
            );

            const executionData = this.helpers.constructExecutionMetaData(
              this.helpers.returnJsonArray(response as IDataObject),
              { itemData: { item: i } },
            );
            returnData.push(...executionData);
          }

          if (operation === 'get') {
            const contactId = this.getNodeParameter('contactId', i) as string;

            const response = await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myServiceApi',
              {
                method: 'GET',
                url: `https://api.myservice.com/v1/contacts/${contactId}`,
              },
            );

            const executionData = this.helpers.constructExecutionMetaData(
              this.helpers.returnJsonArray(response as IDataObject),
              { itemData: { item: i } },
            );
            returnData.push(...executionData);
          }

          if (operation === 'getAll') {
            const returnAll = this.getNodeParameter('returnAll', i) as boolean;

            if (returnAll) {
              // Use a helper that handles pagination (see GenericFunctions.ts)
              const responseData = await myServiceApiRequestAllItems.call(
                this, 'contacts', 'GET', '/contacts',
              );
              const executionData = this.helpers.constructExecutionMetaData(
                this.helpers.returnJsonArray(responseData),
                { itemData: { item: i } },
              );
              returnData.push(...executionData);
            } else {
              const limit = this.getNodeParameter('limit', i) as number;
              const response = await this.helpers.httpRequestWithAuthentication.call(
                this,
                'myServiceApi',
                {
                  method: 'GET',
                  url: 'https://api.myservice.com/v1/contacts',
                  qs: { limit },
                },
              );
              const contacts = (response as IDataObject).data as IDataObject[];
              const executionData = this.helpers.constructExecutionMetaData(
                this.helpers.returnJsonArray(contacts),
                { itemData: { item: i } },
              );
              returnData.push(...executionData);
            }
          }

          if (operation === 'update') {
            const contactId = this.getNodeParameter('contactId', i) as string;
            const additionalFields = this.getNodeParameter(
              'additionalFields', i,
            ) as IDataObject;

            const response = await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myServiceApi',
              {
                method: 'PUT',
                url: `https://api.myservice.com/v1/contacts/${contactId}`,
                body: additionalFields,
              },
            );

            const executionData = this.helpers.constructExecutionMetaData(
              this.helpers.returnJsonArray(response as IDataObject),
              { itemData: { item: i } },
            );
            returnData.push(...executionData);
          }

          if (operation === 'delete') {
            const contactId = this.getNodeParameter('contactId', i) as string;

            await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myServiceApi',
              {
                method: 'DELETE',
                url: `https://api.myservice.com/v1/contacts/${contactId}`,
              },
            );

            const executionData = this.helpers.constructExecutionMetaData(
              this.helpers.returnJsonArray({ deleted: true }),
              { itemData: { item: i } },
            );
            returnData.push(...executionData);
          }
        }
      } catch (error) {
        if (this.continueOnFail()) {
          const executionErrorData = this.helpers.constructExecutionMetaData(
            this.helpers.returnJsonArray({ error: (error as Error).message }),
            { itemData: { item: i } },
          );
          returnData.push(...executionErrorData);
          continue;
        }
        // Always pass itemIndex — the node-operation-error-itemindex lint rule
        // errors on NodeApiError/NodeOperationError thrown in the item loop without it
        throw new NodeApiError(this.getNode(), error as JsonObject, { itemIndex: i });
      }
    }

    return [returnData];
  }
}
```

> **Import note:** `NodeConnectionTypes` (plural) is the const object — use `NodeConnectionTypes.Main` for `inputs`/`outputs`. `NodeConnectionType` (singular) still exists but only as a TypeScript type; using `NodeConnectionType.Main` as a value is a compile error (TS2693) in current n8n-workflow (v2.x).

## The execute() Method

The execute method is called once per node execution. It receives all input items and must return all output items.

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  // Get all items from the previous node
  const items = this.getInputData();

  // Storage for output items
  const returnData: INodeExecutionData[] = [];

  // Get parameters that don't change per item (index 0)
  const resource = this.getNodeParameter('resource', 0) as string;
  const operation = this.getNodeParameter('operation', 0) as string;

  // Loop over each input item
  for (let i = 0; i < items.length; i++) {
    // Get parameters that may change per item (using expression)
    const value = this.getNodeParameter('fieldName', i) as string;

    // Process and push results
    returnData.push({
      json: { /* result data */ },
      pairedItem: { item: i },
    });
  }

  // Return: array of arrays (one per output connector)
  return [returnData];
}
```

### Dynamic Post-Execution Hints

Show a contextual message in the UI after the run finishes — useful when the node ran successfully but something deserves the user's attention (real user: the Split Out node, which warns when a configured field was not found in any input item):

```typescript
// Inside execute(), before returning:
if (fieldNotFoundInAnyItem) {
  this.addExecutionHints({
    message: `The field '${fieldToSplitOut}' wasn't found in any input item`,
    location: 'outputPane',
  });
}

return [returnData];
```

The hint shape is `NodeExecutionHint`: `{ message: string; type?: 'info' | 'warning' | 'danger'; location?: 'outputPane' | 'inputPane' | 'ndv' }`.

> **Migration note:** older guides show wrapping the return value in `new NodeExecutionOutput([returnData], [{ message: '...', location: 'outputPane' }])`. That class no longer exists in current n8n-workflow (v2.x) — use `this.addExecutionHints(...)` instead.

## Item Processing Loop

Key method calls inside the loop:

```typescript
// Get a parameter value (resolves expressions for item i)
const name = this.getNodeParameter('name', i) as string;

// Get optional fields from Additional Fields
const additionalFields = this.getNodeParameter('additionalFields', i) as IDataObject;

// Get a parameter with a default fallback
const limit = this.getNodeParameter('limit', i, 50) as number;

// Access the raw input item
const inputItem = items[i].json;

// Clone input data (never mutate directly)
const newItem = { ...items[i].json };
```

## Error Handling Patterns

### Standard try/catch with continueOnFail

```typescript
for (let i = 0; i < items.length; i++) {
  try {
    // Your logic
  } catch (error) {
    if (this.continueOnFail()) {
      returnData.push({
        json: { error: (error as Error).message },
        pairedItem: { item: i },
      });
      continue;
    }
    // Re-throw as a structured n8n error — include itemIndex (required by the
    // @n8n/community-nodes/node-operation-error-itemindex lint rule inside item loops)
    throw new NodeApiError(this.getNode(), error as JsonObject, { itemIndex: i });
  }
}
```

### Specific HTTP Error Handling

```typescript
try {
  const response = await this.helpers.httpRequestWithAuthentication.call(
    this, 'myApi', options,
  );
} catch (error) {
  if ((error as any).httpCode === '404') {
    throw new NodeApiError(this.getNode(), error as JsonObject, {
      message: 'Resource not found',
      description: `The item with ID "${itemId}" does not exist. Check the ID.`,
    });
  }
  if ((error as any).httpCode === '429') {
    throw new NodeApiError(this.getNode(), error as JsonObject, {
      message: 'Rate limit exceeded',
      description: 'Too many requests. Try again later.',
    });
  }
  throw new NodeApiError(this.getNode(), error as JsonObject);
}
```

### Validation Errors

```typescript
if (!email.includes('@')) {
  throw new NodeOperationError(
    this.getNode(),
    'Invalid email address',
    { itemIndex: i },
  );
}
```

## HTTP Request Patterns

### Basic GET

```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this,
  'myServiceApi',
  {
    method: 'GET',
    url: `https://api.example.com/items/${id}`,

  },
);
```

### POST with Body

```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this,
  'myServiceApi',
  {
    method: 'POST',
    url: 'https://api.example.com/items',
    body: {
      name: 'New Item',
      description: 'A description',
    },

  },
);
```

### GET with Query Parameters

```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this,
  'myServiceApi',
  {
    method: 'GET',
    url: 'https://api.example.com/items',
    qs: {
      page: 1,
      per_page: 50,
      sort: 'created_at',
    },

  },
);
```

### Custom Headers

```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this,
  'myServiceApi',
  {
    method: 'GET',
    url: 'https://api.example.com/items',
    headers: {
      'X-Custom-Header': 'value',
    },

  },
);
```

## Item Linking (constructExecutionMetaData)

Every output item must link back to its source input. The modern approach uses `constructExecutionMetaData`:

```typescript
// Recommended: constructExecutionMetaData (handles pairedItem automatically)
const executionData = this.helpers.constructExecutionMetaData(
  this.helpers.returnJsonArray(responseData),
  { itemData: { item: i } },
);
returnData.push(...executionData);

// Also works for arrays (e.g., getAll returns multiple items):
const results = response.data as IDataObject[];
const executionData = this.helpers.constructExecutionMetaData(
  this.helpers.returnJsonArray(results),
  { itemData: { item: i } },
);
returnData.push(...executionData);
```

Legacy manual approach (still works but less preferred):
```typescript
returnData.push({
  json: response,
  pairedItem: { item: i },
});
```

### pairedItem Forms

```typescript
// Multi-input form — `input` says which input connector the source item came from
// (optional, defaults to 0):
returnData.push({
  json: result,
  pairedItem: { item: i, input: 0 },
});

// Propagating when forwarding items from a previous node — carry the existing
// pairedItem forward instead of re-numbering:
returnData.push({
  json: item.json,
  pairedItem: { item: item.pairedItem as number, input: 0 },
});
```

`pairedItem` may also be an **array** of `{ item, input }` objects when one output item is derived from several input items (e.g. an aggregate or merge).

> **Lint note:** the `@n8n/community-nodes/missing-paired-item` rule (error severity) flags any object literal with a `json` property but no `pairedItem` produced in `execute()` — via `.push()`, `.map()`, or return statements. Objects passed through `constructExecutionMetaData` are exempt because it adds `pairedItem` for you. See `references/validation.md` for the full lint rule catalog.

## Binary Data Handling

For nodes that work with files:

```typescript
// Receiving binary data from previous node:
const binaryData = items[i].binary;
if (binaryData && binaryData.data) {
  const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');
  // Use buffer...
}

// Returning binary data:
const binaryProperty = await this.helpers.prepareBinaryData(
  buffer,
  'filename.pdf',
  'application/pdf',
);

returnData.push({
  json: { success: true },
  binary: { data: binaryProperty },
  pairedItem: { item: i },
});
```

## Trigger Node Patterns

Trigger nodes are always programmatic. They differ from action nodes in several key ways:

| Aspect | Action Node | Trigger Node |
|--------|------------|--------------|
| `group` | `['transform']` | `['trigger']` |
| `inputs` | `[NodeConnectionTypes.Main]` | `[]` (EMPTY) |
| Method | `execute()` | `webhook()`, `poll()`, or `trigger()` |
| Class name | `MyService` | `MyServiceTrigger` |
| File name | `MyService.node.ts` | `MyServiceTrigger.node.ts` |

> **Lint note:** `npm run lint` requires `icon` and `subtitle` on every node description (`require-node-description-fields`) and an explicit `usableAsTool` declaration on every node class (`node-usable-as-tool`). Triggers output Main connections, so they are NOT exempt — declare `usableAsTool: false` on trigger nodes.

### Pattern 1: Webhook Trigger (Auto-Registered)

The external service supports API-based webhook registration. n8n registers/deregisters automatically when the workflow is activated/deactivated.

```typescript
import {
  IHookFunctions, IWebhookFunctions, INodeType,
  INodeTypeDescription, IWebhookResponseData,
  NodeConnectionTypes,
} from 'n8n-workflow';

export class MyServiceTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Service Trigger',
    name: 'myServiceTrigger',
    icon: 'file:myservice.svg',
    group: ['trigger'],
    version: 1,
    subtitle: '={{$parameter["event"]}}',
    description: 'Starts workflow when My Service events occur',
    defaults: { name: 'My Service Trigger' },
    inputs: [],                            // NO inputs for triggers
    outputs: [NodeConnectionTypes.Main],
    usableAsTool: false,
    credentials: [{ name: 'myServiceApi', required: true }],
    webhooks: [
      {
        name: 'default',
        httpMethod: 'POST',
        responseMode: 'onReceived',
        path: 'webhook',
      },
    ],
    properties: [
      {
        displayName: 'Event',
        name: 'event',
        type: 'options',
        options: [
          { name: 'Contact Created', value: 'contact.created' },
          { name: 'Deal Updated', value: 'deal.updated' },
        ],
        default: 'contact.created',
        required: true,
      },
    ],
  };

  webhookMethods = {
    default: {
      async checkExists(this: IHookFunctions): Promise<boolean> {
        const webhookData = this.getWorkflowStaticData('node');
        if (webhookData.webhookId) {
          return true;
        }
        return false;
      },

      async create(this: IHookFunctions): Promise<boolean> {
        const webhookData = this.getWorkflowStaticData('node');
        const webhookUrl = this.getNodeWebhookUrl('default');
        const event = this.getNodeParameter('event') as string;

        // Register webhook with external API
        const response = await myServiceApiRequest.call(this, 'POST', '/webhooks', {
          url: webhookUrl,
          events: [event],
        });

        // Store registration ID for cleanup
        webhookData.webhookId = response.id;
        return true;
      },

      async delete(this: IHookFunctions): Promise<boolean> {
        const webhookData = this.getWorkflowStaticData('node');
        const webhookId = webhookData.webhookId as string;

        if (webhookId) {
          await myServiceApiRequest.call(this, 'DELETE', `/webhooks/${webhookId}`);
          delete webhookData.webhookId;
        }
        return true;
      },
    },
  };

  async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const bodyData = this.getBodyData();
    return {
      workflowData: [this.helpers.returnJsonArray(bodyData)],
    };
  }
}
```

**Lifecycle:**
1. **Workflow activated** → `checkExists()` → if false → `create()` registers webhook
2. **Event received** → `webhook()` processes payload → triggers workflow
3. **Workflow deactivated** → `delete()` removes webhook registration

Use `this.getWorkflowStaticData('node')` to persist data (webhook IDs) between lifecycle calls. This data survives n8n restarts.

### Pattern 2: Simple Webhook (Manual URL)

No auto-registration — the user manually configures the webhook URL in the external service.

```typescript
export class MyServiceTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Service Trigger',
    name: 'myServiceTrigger',
    icon: 'file:myservice.svg',
    group: ['trigger'],
    version: 1,
    subtitle: 'webhook',
    description: 'Starts workflow on My Service webhook',
    defaults: { name: 'My Service Trigger' },
    inputs: [],
    outputs: [NodeConnectionTypes.Main],
    usableAsTool: false,
    webhooks: [
      {
        name: 'default',
        httpMethod: 'POST',
        responseMode: 'onReceived',
        path: 'webhook',
      },
    ],
    properties: [],
  };

  // CAUTION: no webhookMethods — user configures the URL manually (see caveat below)
  async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const bodyData = this.getBodyData();
    return {
      workflowData: [this.helpers.returnJsonArray(bodyData)],
    };
  }
}
```

> **Lint caveat:** under the current `@n8n/eslint-plugin-community-nodes` linter, a node that declares `webhooks` but has no `webhookMethods` class property fails the `webhook-lifecycle-complete` rule (error severity), and the n8n verification scan runs the same rule set with inline disables ignored. For community nodes, either implement the `checkExists`/`create`/`delete` lifecycle (Pattern 1) — or accept that this manual-URL pattern fails `npm run lint` and the verification scan as-is.

### Pattern 3: Poll Trigger

n8n polls the external API on a schedule, checking for new data since the last poll.

```typescript
import {
  IPollFunctions, INodeType, INodeTypeDescription,
  INodeExecutionData, NodeConnectionTypes,
} from 'n8n-workflow';

export class MyServiceTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Service Trigger',
    name: 'myServiceTrigger',
    icon: 'file:myservice.svg',
    group: ['trigger'],
    version: 1,
    subtitle: 'new items',
    description: 'Starts workflow when new items appear',
    defaults: { name: 'My Service Trigger' },
    inputs: [],
    outputs: [NodeConnectionTypes.Main],
    usableAsTool: false,
    polling: true,                         // REQUIRED for poll triggers
    credentials: [{ name: 'myServiceApi', required: true }],
    properties: [],
  };

  async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const webhookData = this.getWorkflowStaticData('node');
    const lastChecked = webhookData.lastTimeChecked as string
      || new Date().toISOString();

    // Query API for items created/updated since lastChecked
    const items = await myServiceApiRequest.call(
      this, 'GET', '/items', {}, { since: lastChecked },
    );

    // Update last checked time
    webhookData.lastTimeChecked = new Date().toISOString();

    if (items.length === 0) {
      return null;                         // Return null = no new data, don't trigger
    }

    return [this.helpers.returnJsonArray(items)];
  }
}
```

**Key points:**
- Set `polling: true` in the description
- `poll()` returns `INodeExecutionData[][] | null` — return `null` when there are no new items
- Track state with `getWorkflowStaticData('node')` to avoid re-processing old data

### Pattern 4: Event/Stream Trigger

Long-running listener for message queues, SSE, or WebSocket connections:

```typescript
import {
  ITriggerFunctions, ITriggerResponse, INodeType,
  INodeTypeDescription, NodeConnectionTypes,
} from 'n8n-workflow';

export class MyServiceTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Service Trigger',
    name: 'myServiceTrigger',
    icon: 'file:myservice.svg',
    group: ['trigger'],
    version: 1,
    subtitle: '={{$parameter["event"]}}',
    description: 'Starts workflow on My Service events',
    defaults: { name: 'My Service Trigger' },
    inputs: [],
    outputs: [NodeConnectionTypes.Main],
    usableAsTool: false,
    properties: [/* ... */],
  };

  async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    // Set up listener
    const client = new MyServiceClient(/* ... */);
    client.on('message', (data) => {
      this.emit([this.helpers.returnJsonArray(data)]);
    });
    await client.connect();

    // Return cleanup + manual trigger functions
    const closeFunction = async () => {
      await client.disconnect();
    };
    const manualTriggerFunction = async () => {
      // Emit test data for manual workflow runs
      this.emit([this.helpers.returnJsonArray({ test: true })]);
    };

    return { closeFunction, manualTriggerFunction };
  }
}
```

**Key points:**
- `this.emit()` pushes data into the workflow whenever an event arrives
- `closeFunction` runs on workflow deactivation — clean up connections
- `manualTriggerFunction` provides test data when user clicks "Test Workflow"

### Trigger Common Mistakes

| Mistake | Fix |
|---------|-----|
| Trigger node with `inputs: [NodeConnectionTypes.Main]` | Use `inputs: []` — triggers have NO inputs |
| Missing `polling: true` on poll trigger | Required for n8n to schedule the poll |
| Not storing webhook ID in static data | Use `getWorkflowStaticData('node')` to persist between restarts |
| `poll()` returning empty array instead of null | Return `null` for no new data (empty array triggers with no items) |
| Using `IExecuteFunctions` in trigger | Use `IWebhookFunctions`, `IPollFunctions`, or `ITriggerFunctions` |
| Forgetting `delete()` in webhookMethods | Always clean up webhook registrations on deactivation |

## Full Versioning Structure

When you need fundamentally different behavior between versions:

```
nodes/MyNode/
├── MyNode.node.ts          # Version router
├── v1/
│   ├── MyNodeV1.node.ts    # INodeType for v1
│   └── actions/
└── v2/
    ├── MyNodeV2.node.ts    # INodeType for v2
    └── actions/
```

The main file:

```typescript
import type { INodeTypeBaseDescription, IVersionedNodeType } from 'n8n-workflow';
import { VersionedNodeType } from 'n8n-workflow';
import { MyNodeV1 } from './v1/MyNodeV1.node';
import { MyNodeV2 } from './v2/MyNodeV2.node';

export class MyNode extends VersionedNodeType {
  constructor() {
    const baseDescription: INodeTypeBaseDescription = {
      displayName: 'My Node',
      name: 'myNode',
      icon: 'file:myNode.svg',
      group: ['transform'],
      description: 'Interact with My Service',
      defaultVersion: 2,
    };

    const nodeVersions: IVersionedNodeType['nodeVersions'] = {
      1: new MyNodeV1(baseDescription),
      2: new MyNodeV2(baseDescription),
    };

    super(nodeVersions, baseDescription);
  }
}
```

### Feature-Based Versioning

Instead of splitting into separate version classes, a single node can gate functionality by named feature flags. Declare a `features` object on the description — each key maps a feature name to a `'@version'` condition (evaluated against the node's `typeVersion`):

```typescript
description: INodeTypeDescription = {
  // ...
  version: [1, 1.1, 1.2],
  defaultVersion: 1.2,
  features: {
    bulkOperations: { '@version': [{ _cnd: { gte: 1.2 } }] },
  },
  properties: [
    {
      displayName: 'Batch Size',
      name: 'batchSize',
      type: 'number',
      default: 100,
      displayOptions: {
        show: { '@feature': ['bulkOperations'] },  // Only shown when the feature is enabled
      },
    },
  ],
};
```

- `features` is typed as `Record<string, { '@version': Array<number | DisplayCondition> }>` (`NodeFeaturesDefinition` in n8n-workflow).
- In `displayOptions`, `show: { '@feature': ['featureName'] }` matches against the list of features enabled for the current node version.
- At runtime, check inside `execute()` with `this.isNodeFeatureEnabled('bulkOperations'): boolean`.

```typescript
// In execute():
if (this.isNodeFeatureEnabled('bulkOperations')) {
  // Behavior only for typeVersion >= 1.2
}
```

The benefit over `'@version'` checks scattered through properties and code: the version-to-capability mapping lives in one place, and renaming or re-versioning a feature is a one-line change.

## GenericFunctions.ts Pattern

Create a shared helper file to keep your execute method clean:

```typescript
// nodes/MyService/GenericFunctions.ts
import type {
  IExecuteFunctions,
  IHookFunctions,
  ILoadOptionsFunctions,
  IPollFunctions,
  IWebhookFunctions,
  IHttpRequestMethods,
  IHttpRequestOptions,
  IDataObject,
  JsonObject,
} from 'n8n-workflow';
import { NodeApiError } from 'n8n-workflow';

const BASE_URL = 'https://api.myservice.com/v1';

export async function myServiceApiRequest(
  this: IExecuteFunctions | IHookFunctions | ILoadOptionsFunctions | IPollFunctions | IWebhookFunctions,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
  uri?: string,                                  // Optional absolute URL override
  option: Partial<IHttpRequestOptions> = {},     // Extra request options (standard nodes-base pattern)
): Promise<any> {
  const options: IHttpRequestOptions = {
    method,
    url: uri ?? `${BASE_URL}${endpoint}`,
    body,
    qs,
    ...option,
  };
  try {
    return await this.helpers.httpRequestWithAuthentication.call(this, 'myServiceApi', options);
  } catch (error) {
    throw new NodeApiError(this.getNode(), error as JsonObject);
  }
}

export async function myServiceApiRequestAllItems(
  this: IExecuteFunctions | IHookFunctions | ILoadOptionsFunctions,
  propertyName: string,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
): Promise<IDataObject[]> {
  const returnData: IDataObject[] = [];
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await myServiceApiRequest.call(
      this, method, endpoint, body, { ...qs, page, per_page: 100 },
    );
    const items = response[propertyName] as IDataObject[];
    returnData.push(...items);
    hasMore = items.length === 100;
    page++;
  }

  return returnData;
}
```

### Pagination Variants

The `apiRequestAllItems` function above uses page-number pagination. Here are the other common patterns:

**Cursor-based pagination:**
```typescript
export async function myServiceApiRequestAllItems(
  this: IExecuteFunctions,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
): Promise<IDataObject[]> {
  const returnData: IDataObject[] = [];
  let cursor: string | undefined;

  do {
    if (cursor) qs.cursor = cursor;
    const response = await myServiceApiRequest.call(this, method, endpoint, body, qs);
    returnData.push(...response.results);
    cursor = response.next_cursor;
  } while (cursor);

  return returnData;
}
```

**Offset-based pagination:**
```typescript
export async function myServiceApiRequestAllItems(
  this: IExecuteFunctions,
  propertyName: string,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
): Promise<IDataObject[]> {
  const returnData: IDataObject[] = [];
  qs.limit = 100;
  qs.offset = 0;
  let responseItems: IDataObject[];

  do {
    const response = await myServiceApiRequest.call(this, method, endpoint, body, qs);
    responseItems = response[propertyName] as IDataObject[];
    returnData.push(...responseItems);
    qs.offset = (qs.offset as number) + (qs.limit as number);
  } while (responseItems.length === qs.limit);

  return returnData;
}
```

Then use in your node:
```typescript
import { myServiceApiRequest, myServiceApiRequestAllItems } from './GenericFunctions';

// In execute():
const response = await myServiceApiRequest.call(this, 'POST', '/contacts', body);
```

## Complete `this.*` Methods Reference

All methods available on `this` inside `execute()` (from `IExecuteFunctions`):

### Parameter & Input Methods

```typescript
// Get parameter value (resolves expressions per item). Type-safe overloads exist for string, number, boolean, IDataObject.
this.getNodeParameter(parameterName: string, itemIndex: number, fallbackValue?: any, options?: IGetNodeParameterOptions): NodeParameterValueType | object;

// IGetNodeParameterOptions:
// {
//   ensureType?: 'string' | 'number' | 'boolean' | 'object' | 'array' | 'json';  // Auto-convert
//   extractValue?: boolean;    // Extract value from resourceLocator
//   rawExpressions?: boolean;  // Get raw unresolved expressions
//   skipValidation?: boolean;  // Skip parameter validation
// }

// Get all input items
this.getInputData(inputIndex?: number, connectionType?: NodeConnectionType): INodeExecutionData[];

// Get input from AI/LangChain connections
this.getInputConnectionData(connectionType: AINodeConnectionType, itemIndex: number, inputIndex?: number): Promise<unknown>;

// Track input data lineage
this.getInputSourceData(inputIndex?: number, connectionType?: NodeConnectionType): ISourceData;
```

### Node & Workflow Information

```typescript
this.getNode(): INode;                          // { id, name, typeVersion, type, position, disabled?, parameters, credentials? }
this.getWorkflow(): IWorkflowMetadata;           // { id?, name?, active }
this.getWorkflowSettings(): IWorkflowSettings;   // { timezone?, errorWorkflow?, saveDataErrorExecution?, ... }
this.getTimezone(): string;
this.getRestApiUrl(): string;
this.getInstanceBaseUrl(): string;
this.getInstanceId(): string;
this.getExecutionId(): string;
```

### Credentials

```typescript
// Get decrypted credentials — supports generics for type safety
this.getCredentials<T extends object = ICredentialDataDecryptedObject>(type: string, itemIndex?: number): Promise<T>;

// Example:
const creds = await this.getCredentials<{ apiKey: string; baseUrl: string }>('myServiceApi');
```

### Execution State & Control

```typescript
this.continueOnFail(): boolean;                   // Check if user enabled "Continue On Fail"
this.getMode(): WorkflowExecuteMode;              // 'manual' | 'trigger' | 'webhook' | 'cli' | 'retry' | 'internal' | 'error' | 'integrated' | 'evaluation' | 'chat' | 'agent'
this.evaluateExpression(expression: string, itemIndex: number): NodeParameterValueType;  // Evaluate n8n expressions dynamically
this.getWorkflowDataProxy(itemIndex: number): IWorkflowDataProxyData;  // Access $json, $items(), $node, $parameter, $env, etc.
this.getExecuteData(): IExecuteData;
this.getExecutionCancelSignal(): AbortSignal | undefined;
this.onExecutionCancellation(handler: () => unknown): void;
this.isToolExecution(): boolean;                   // True when running as AI agent tool
this.isStreaming(): boolean;
```

### Persistent Storage

```typescript
// Persist data across executions (survives restarts). Used by triggers for webhook IDs, poll cursors, etc.
this.getWorkflowStaticData(type: 'node' | 'global'): IDataObject;
//   'node'   → scoped to this node instance
//   'global' → shared across all nodes in the workflow

// Share data between nodes within a single execution (in-memory only)
this.getContext(type: 'flow' | 'node'): IContextObject;
//   'flow'  → shared across all nodes in this execution
//   'node'  → private to this node in this execution
```

### Communication & Output

```typescript
this.sendMessageToUI(message: any): void;          // Send status updates to the n8n UI
this.sendResponse(response: IExecuteResponsePromiseData): void;
this.addExecutionHints(...hints: NodeExecutionHint[]): void;  // Add warnings/info shown after execution

// Node graph navigation
this.getChildNodes(nodeName: string, options?: { includeNodeParameters?: boolean }): NodeTypeAndVersion[];
this.getParentNodes(nodeName: string, options?: { includeNodeParameters?: boolean; connectionType?: NodeConnectionType; depth?: number }): NodeTypeAndVersion[];
this.getNodeInputs(): INodeInputConfiguration[];
this.getNodeOutputs(): INodeOutputConfiguration[];
```

### Sub-Workflow Execution

```typescript
this.executeWorkflow(
  workflowInfo: { id?: string; code?: IWorkflowBase },
  inputData?: INodeExecutionData[],
  parentCallbackManager?: CallbackManager,
  options?: {
    doNotWaitToFinish?: boolean;
    parentExecution?: RelatedExecution;
    executionMode?: WorkflowExecuteMode;
  },
): Promise<ExecuteWorkflowData>;

this.putExecutionToWait(waitTill: Date): Promise<void>;  // Pause execution until a date
```

### Logging

```typescript
this.logger: Logger;  // Available for logging: this.logger.info(), this.logger.warn(), this.logger.error()
```

> **Deprecation note:** `this.helpers.requestWithAuthentication` and `IRequestOptions` are **deprecated**. Always use `httpRequestWithAuthentication` with `IHttpRequestOptions`. The new interface uses `url` (not `uri`) and defaults to JSON parsing (no `json: true` needed).

---

## Complete `this.helpers` Reference

The `helpers` object combines multiple interfaces:

```typescript
this.helpers: RequestHelperFunctions & BaseHelperFunctions & BinaryHelperFunctions &
              DeduplicationHelperFunctions & FileSystemHelperFunctions &
              SSHTunnelFunctions & DataTableProxyFunctions & { /* additional */ }
```

### Request Helpers

```typescript
// PRIMARY — use these for all HTTP requests
this.helpers.httpRequest(requestOptions: IHttpRequestOptions): Promise<any>;
this.helpers.httpRequestWithAuthentication(
  credentialsType: string,
  requestOptions: IHttpRequestOptions,
  additionalCredentialOptions?: IAdditionalCredentialOptions,
): Promise<any>;

// Paginated requests (declarative-style pagination)
this.helpers.requestWithAuthenticationPaginated(
  requestOptions: IRequestOptions,
  itemIndex: number,
  paginationOptions: PaginationOptions,
  credentialsType?: string,
): Promise<any[]>;

// DEPRECATED — avoid in new code
this.helpers.request(uriOrObject: string | IRequestOptions, options?: IRequestOptions): Promise<any>;
this.helpers.requestWithAuthentication(credentialsType: string, requestOptions: IRequestOptions, ...): Promise<any>;
this.helpers.requestOAuth1(credentialsType: string, requestOptions: IRequestOptions): Promise<any>;
this.helpers.requestOAuth2(credentialsType: string, requestOptions: IRequestOptions, oAuth2Options?: IOAuth2Options): Promise<any>;
```

### Data Helpers

```typescript
// Wrap raw JSON as INodeExecutionData[] (handles both single object and arrays)
this.helpers.returnJsonArray(jsonData: IDataObject | IDataObject[]): INodeExecutionData[];

// Attach pairedItem metadata to output items (preferred over manual pairedItem)
this.helpers.constructExecutionMetaData(
  inputData: INodeExecutionData[],
  options: { itemData: IPairedItemData | IPairedItemData[] },
): NodeExecutionWithMetadata[];

// Normalize inconsistent item formats (auto-wraps in { json: ... } if needed)
this.helpers.normalizeItems(items: INodeExecutionData | INodeExecutionData[]): INodeExecutionData[];

// Copy specific properties from input items
this.helpers.copyInputItems(items: INodeExecutionData[], properties: string[]): IDataObject[];

// Create a deferred promise
this.helpers.createDeferredPromise<T = void>(): IDeferredPromise<T>;
```

### Binary Data Helpers

```typescript
// Convert Buffer/Stream to n8n binary format (auto-detects MIME type and extension)
this.helpers.prepareBinaryData(binaryData: Buffer | Readable, filePath?: string, mimeType?: string): Promise<IBinaryData>;

// Store binary data buffer
this.helpers.setBinaryDataBuffer(data: IBinaryData, binaryData: Buffer): Promise<IBinaryData>;

// Get binary data as Buffer from an input item
this.helpers.getBinaryDataBuffer(itemIndex: number, propertyName: string): Promise<Buffer>;

// Assert binary data exists on an input item (throws helpful error if not found)
this.helpers.assertBinaryData(itemIndex: number, propertyName: string): IBinaryData;

// Convert Buffer/Stream to Buffer or String
this.helpers.binaryToBuffer(body: Buffer | Readable): Promise<Buffer>;
this.helpers.binaryToString(body: Buffer | Readable, encoding?: BufferEncoding): Promise<string>;

// Detect encoding of a binary buffer
this.helpers.detectBinaryEncoding(buffer: Buffer): string;

// Streaming access for large files
this.helpers.getBinaryPath(binaryDataId: string): string;
this.helpers.getBinaryStream(binaryDataId: string, chunkSize?: number): Promise<Readable>;
this.helpers.getBinaryMetadata(binaryDataId: string): Promise<{ fileName?: string; mimeType?: string; fileSize: number }>;

// Create a signed URL for binary data
this.helpers.createBinarySignedUrl(binaryData: IBinaryData, expiresIn?: string): string;
```

**IBinaryData type:**
```typescript
interface IBinaryData {
  data: string;                    // Base64-encoded content (or empty if stored externally)
  mimeType: string;
  fileType?: 'text' | 'json' | 'image' | 'audio' | 'video' | 'pdf' | 'html';
  fileName?: string;
  directory?: string;
  fileExtension?: string;
  fileSize?: string;               // Human-readable
  bytes?: number;
  id?: string;                     // Present when stored in external binary storage
}
```

### FileSystem Helpers

```typescript
// Resolve and validate a file path (MUST call before any file operation)
this.helpers.resolvePath(path: PathLike): Promise<ResolvedFilePath>;

// Check if a path is blocked by security settings
this.helpers.isFilePathBlocked(filePath: ResolvedFilePath): boolean;

// Create a readable stream for a file
this.helpers.createReadStream(filePath: ResolvedFilePath): Promise<Readable>;

// Get the node's storage directory
this.helpers.getStoragePath(): string;

// Write content to a file
this.helpers.writeContentToFile(path: ResolvedFilePath, content: string | Buffer | Readable, flag?: number): Promise<void>;
```

**Usage pattern:**
```typescript
// Always resolve path before any file operation
const resolvedPath = await this.helpers.resolvePath('/path/to/file.txt');
if (!this.helpers.isFilePathBlocked(resolvedPath)) {
  const stream = await this.helpers.createReadStream(resolvedPath);
  // ...
}
```

### Deduplication Helpers

Track processed items across executions to prevent duplicates (useful in trigger/poll nodes):

```typescript
// Check which items are new and record them
this.helpers.checkProcessedAndRecord(
  items: (string | number)[],           // Item identifiers
  scope: 'node' | 'workflow',
  options: { mode: 'entries' | 'latestIncrementalKey' | 'latestDate'; maxEntries?: number },
): Promise<IDeduplicationOutput>;       // { new: [...], processed: [...] }

// Same but for full IDataObject items using a specific property
this.helpers.checkProcessedItemsAndRecord(
  propertyName: string,
  items: IDataObject[],
  scope: 'node' | 'workflow',
  options: ICheckProcessedOptions,
): Promise<IDeduplicationOutputItems>;

// Remove items from the processed list
this.helpers.removeProcessed(items: (string | number)[], scope: 'node' | 'workflow', options: ICheckProcessedOptions): Promise<void>;

// Clear all tracked items
this.helpers.clearAllProcessedItems(scope: 'node' | 'workflow', options: ICheckProcessedOptions): Promise<void>;

// Count tracked items
this.helpers.getProcessedDataCount(scope: 'node' | 'workflow', options: ICheckProcessedOptions): Promise<number>;
```

### SSH Tunnel Helpers

```typescript
this.helpers.getSSHClient(credentials: SSHCredentials, abortController?: AbortController): Promise<SSHClient>;
this.helpers.updateLastUsed(client: SSHClient): void;
```

### nodeHelpers

```typescript
// Copy a file on disk into n8n binary data format
this.nodeHelpers.copyBinaryFile(filePath: string, fileName: string, mimeType?: string): Promise<IBinaryData>;
```

---

## Dynamic Options (loadOptions & listSearch)

Populate dropdown options dynamically by fetching from an API. Define a `methods` property on the node class:

```typescript
export class MyService implements INodeType {
  description: INodeTypeDescription = { /* ... */ };

  methods = {
    loadOptions: {
      // Function name matches the property's `typeOptions.loadOptionsMethod`
      async getUsers(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
        const response = await myServiceApiRequest.call(this, 'GET', '/users');
        return (response.users as IDataObject[]).map((user) => ({
          name: user.name as string,
          value: user.id as string,
        }));
      },

      async getProjects(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
        const response = await myServiceApiRequest.call(this, 'GET', '/projects');
        return (response.projects as IDataObject[])
          .map((p) => ({ name: p.title as string, value: p.id as string }))
          .sort((a, b) => a.name.localeCompare(b.name));
      },
    },

    // Searchable dropdowns with pagination
    listSearch: {
      async searchContacts(
        this: ILoadOptionsFunctions,
        filter?: string,
        paginationToken?: string,
      ): Promise<INodeListSearchResult> {
        const qs: IDataObject = { limit: 20 };
        if (filter) qs.search = filter;
        if (paginationToken) qs.cursor = paginationToken;

        const response = await myServiceApiRequest.call(this, 'GET', '/contacts', {}, qs);
        return {
          results: (response.data as IDataObject[]).map((c) => ({
            name: c.name as string,
            value: c.id as string,
            url: c.url as string,       // Optional: shown as link in UI
          })),
          paginationToken: response.next_cursor || undefined,
        };
      },
    },

    // Resource mapping for dynamic field schemas
    resourceMapping: {
      async getColumns(this: ILoadOptionsFunctions): Promise<ResourceMapperFields> {
        const tableId = this.getNodeParameter('tableId') as string;
        const response = await myServiceApiRequest.call(this, 'GET', `/tables/${tableId}/columns`);
        return {
          // ResourceMapperField requires `display` (false hides the field entirely)
          // and `type` must be a FieldType:
          // 'string' | 'number' | 'boolean' | 'dateTime' | 'time' | 'array' | 'object' | 'options'
          fields: (response.columns as IDataObject[]).map((col) => ({
            id: col.id as string,
            displayName: col.name as string,
            required: col.required as boolean,
            defaultMatch: col.isPrimary as boolean,
            display: true,
            canBeUsedToMatch: true,
            type: 'string',
          })),
        };
      },
    },
  };

  async execute(this: IExecuteFunctions) { /* ... */ }
}
```

**Wire properties to loadOptions:**

```typescript
{
  displayName: 'User',
  name: 'userId',
  type: 'options',
  typeOptions: {
    loadOptionsMethod: 'getUsers',     // Must match the method name above
  },
  default: '',
}

// For listSearch:
{
  displayName: 'Contact',
  name: 'contactId',
  type: 'resourceLocator',
  default: { mode: 'list', value: '' },
  modes: [
    {
      displayName: 'From List',
      name: 'list',
      type: 'list',
      typeOptions: {
        searchListMethod: 'searchContacts',
        searchable: true,
      },
    },
    {
      displayName: 'By ID',
      name: 'id',
      type: 'string',
    },
  ],
}
```

**`ILoadOptionsFunctions` extra methods** (not available in `IExecuteFunctions`):
```typescript
// Get the value of the parameter currently being edited in the UI
this.getCurrentNodeParameter(parameterName: string): NodeParameterValueType | object | undefined;

// Get all current parameter values
this.getCurrentNodeParameters(): INodeParameters | undefined;
```

---

## Multiple Outputs

Nodes can route items to different outputs. The return value is `INodeExecutionData[][]` — one array per output:

```typescript
export class MyRouter implements INodeType {
  description: INodeTypeDescription = {
    // ...
    outputs: [NodeConnectionTypes.Main, NodeConnectionTypes.Main],  // Two outputs
    outputNames: ['Matched', 'Unmatched'],
    // ...
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const matched: INodeExecutionData[] = [];
    const unmatched: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
      const value = this.getNodeParameter('field', i) as string;
      if (value) {
        matched.push({ json: items[i].json, pairedItem: { item: i } });
      } else {
        unmatched.push({ json: items[i].json, pairedItem: { item: i } });
      }
    }

    return [matched, unmatched];  // Index 0 → "Matched" output, Index 1 → "Unmatched"
  }
}
```

**Dynamic outputs with expressions** (advanced — used by Switch node):
```typescript
outputs: `={{
  ((parameters) => {
    const rules = parameters.rules?.rules ?? [];
    return rules.map(value => ({
      type: "${NodeConnectionTypes.Main}",
      displayName: value.outputKey
    }));
  })($parameter)
}}`,
```

---

## Execution Control

### Execute Sub-Workflows

```typescript
// Trigger another workflow by ID
const result = await this.executeWorkflow(
  { id: 'workflow-id-here' },
  [items[i]],                           // Input data to pass
  undefined,
  {
    doNotWaitToFinish: false,            // Set true for fire-and-forget
    parentExecution: {
      executionId: this.getExecutionId(),
      workflowId: this.getWorkflow().id!,
    },
  },
);

// result.data contains the sub-workflow output
const outputItems = result.data?.[0] ?? [];
```

### Wait/Resume Pattern

```typescript
// Pause execution until a specific date (e.g., for approval workflows)
await this.putExecutionToWait(new Date('2024-12-31'));

// Generate a signed URL that can resume this execution from an external callback
const resumeUrl = this.getSignedResumeUrl({ action: 'approved' });
```

### Cancellation Handling

```typescript
const signal = this.getExecutionCancelSignal();

// Pass to fetch/axios for automatic cancellation
const response = await fetch(url, { signal });

// Or register cleanup handler
this.onExecutionCancellation(() => {
  client.disconnect();
});
```

---

## Advanced Binary Data Patterns

### Download File from API → Return as Binary

```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'myServiceApi', {
    method: 'GET',
    url: `https://api.example.com/files/${fileId}/download`,
    encoding: 'arraybuffer',       // Critical: get raw binary, not string
    returnFullResponse: true,
  },
);

const binaryData = await this.helpers.prepareBinaryData(
  Buffer.from(response.body),
  fileName,                          // Optional: auto-detected if omitted
  response.headers['content-type'],  // Optional: auto-detected from buffer
);

returnData.push({
  json: { fileId, fileName },
  binary: { data: binaryData },
  pairedItem: { item: i },
});
```

### Upload Binary Data (Multipart Form-Data)

```typescript
const binaryData = this.helpers.assertBinaryData(i, 'data');

let fileContent: Buffer | Readable;
if (binaryData.id) {
  // External storage — use stream for efficiency
  fileContent = await this.helpers.getBinaryStream(binaryData.id);
} else {
  // Inline base64 — convert to buffer
  fileContent = Buffer.from(binaryData.data, 'base64');
}

const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'myServiceApi', {
    method: 'POST',
    url: 'https://api.example.com/files/upload',
    body: fileContent,
    headers: {
      'Content-Type': binaryData.mimeType ?? 'application/octet-stream',
    },
  },
);
```

### Upload via FormData (Multipart)

Use the built-in global `FormData` (Node 18+/undici) — no import needed:

```typescript
const binaryData = this.helpers.assertBinaryData(i, 'data');
const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');

const formData = new FormData();
formData.append(
  'file',
  new Blob([buffer], { type: binaryData.mimeType }),
  binaryData.fileName,
);
formData.append('name', fileName);

const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'myServiceApi', {
    method: 'POST',
    url: 'https://api.example.com/files',
    body: formData,
    // No manual getHeaders() — a FormData body auto-sets multipart/form-data
  },
);
```

> **Caveat:** never `import FormData from 'form-data'` — the npm package is not on the import allowlist (`@n8n/community-nodes/no-restricted-imports`, error) and a runtime dependency also fails the `no-runtime-dependencies` requirement for verified nodes.

### Download Attachments into Binary Properties

```typescript
// When an API returns multiple file URLs in its response
const attachments = response.attachments as Array<{ url: string; filename: string; mimeType: string }>;
const binaryObject: IBinaryKeyData = {};

for (const [index, attachment] of attachments.entries()) {
  const fileBuffer = await this.helpers.httpRequest({
    method: 'GET',
    url: attachment.url,
    encoding: 'arraybuffer',
  });

  binaryObject[`file_${index}`] = await this.helpers.prepareBinaryData(
    Buffer.from(fileBuffer),
    attachment.filename,
    attachment.mimeType,
  );
}

returnData.push({
  json: response,
  binary: binaryObject,
  pairedItem: { item: i },
});
```

---

## Advanced GenericFunctions Patterns

### Rate Limiting with Retry

Real pattern from Slack integration — respects `Retry-After` header. Use `sleep` from n8n-workflow, never `setTimeout` (a restricted global under `@n8n/community-nodes/no-restricted-globals` — using it makes the package ineligible for n8n Cloud verification):

```typescript
import { sleep } from 'n8n-workflow';

export async function myServiceApiRequestWithRetry(
  this: IExecuteFunctions,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
  maxRetries = 3,
): Promise<any> {
  let retryCount = 0;

  while (true) {
    try {
      return await myServiceApiRequest.call(this, method, endpoint, body, qs);
    } catch (error) {
      if ((error as any).httpCode === '429' && retryCount < maxRetries) {
        const retryAfter = (error as any).headers?.['retry-after'];
        const waitMs = retryAfter ? parseInt(retryAfter, 10) * 1000 : 30_000;
        await sleep(waitMs);
        retryCount++;
        continue;
      }
      throw error;
    }
  }
}
```

### Link-Header Pagination

Used by GitHub, Shopify, and similar APIs. Requires the extended helper signature from the GenericFunctions.ts template above (`uri?: string` override plus `option: Partial<IHttpRequestOptions>`). Use `returnFullResponse: true` to get `{ body, headers, statusCode, statusMessage }` — the old `resolveWithFullResponse` belongs to the deprecated `IRequestOptions` interface:

```typescript
export async function myServiceApiRequestAllItems(
  this: IExecuteFunctions,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
): Promise<IDataObject[]> {
  const returnData: IDataObject[] = [];
  let uri: string | undefined;

  do {
    const response = await myServiceApiRequest.call(
      this, method, endpoint, body, qs, uri, { returnFullResponse: true },
    );

    returnData.push(...(response.body as IDataObject[]));

    // Parse Link header: <url>; rel="next"
    const linkHeader = response.headers?.link as string | undefined;
    if (linkHeader?.includes('rel="next"')) {
      uri = linkHeader.split(';')[0].replace('<', '').replace('>', '').trim();
    } else {
      uri = undefined;
    }
  } while (uri);

  return returnData;
}
```

### Multiple Authentication Methods

Common pattern when a service supports both API key and OAuth2:

```typescript
export async function myServiceApiRequest(
  this: IExecuteFunctions | ILoadOptionsFunctions | IHookFunctions | IWebhookFunctions,
  method: IHttpRequestMethods,
  endpoint: string,
  body: IDataObject = {},
  qs: IDataObject = {},
): Promise<any> {
  const options: IHttpRequestOptions = {
    method,
    url: `https://api.myservice.com/v1${endpoint}`,
    body,
    qs,
  };

  const authenticationMethod = this.getNodeParameter('authentication', 0) as string;

  try {
    if (authenticationMethod === 'apiKey') {
      return await this.helpers.httpRequestWithAuthentication.call(this, 'myServiceApi', options);
    } else {
      return await this.helpers.httpRequestWithAuthentication.call(this, 'myServiceOAuth2Api', options);
    }
  } catch (error) {
    throw new NodeApiError(this.getNode(), error as JsonObject);
  }
}
```

### Data Transformation Helpers

Common utility functions found across GenericFunctions.ts files:

```typescript
// Validate JSON strings (used before passing user-provided JSON to APIs)
export function validateJSON(json: string | undefined): any {
  try {
    return JSON.parse(json!);
  } catch {
    return undefined;
  }
}

// Remove null/undefined/empty values from request bodies
export function clean(obj: IDataObject): IDataObject {
  for (const key in obj) {
    if (obj[key] === null || obj[key] === undefined || obj[key] === '') {
      delete obj[key];
    }
  }
  return obj;
}

// Convert camelCase keys to snake_case (for APIs that expect it)
import { snakeCase } from 'lodash';

export function keysToSnakeCase(obj: IDataObject): IDataObject {
  const result: IDataObject = {};
  for (const key of Object.keys(obj)) {
    result[snakeCase(key)] = obj[key];
  }
  return result;
}

// Simplify nested API responses (flatten metadata)
export function simplifyResponse(data: IDataObject[], propertyName: string): IDataObject[] {
  return data.map((item) => {
    const nested = item[propertyName] as IDataObject;
    delete item[propertyName];
    return { ...item, ...nested };
  });
}
```

### Logging in GenericFunctions

```typescript
export async function myServiceApiRequest(
  this: IExecuteFunctions | ILoadOptionsFunctions,
  // ...
): Promise<any> {
  // ...
  } catch (error) {
    const workflow = this.getWorkflow();
    const node = this.getNode();
    this.logger.error(
      `API error in '${node.name}' (workflow '${workflow.id}'): ${(error as Error).message}`,
      { node: node.name, workflowId: workflow.id },
    );
    throw new NodeApiError(this.getNode(), error as JsonObject);
  }
}
```
