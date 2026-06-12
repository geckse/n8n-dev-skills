# AI Sub-Nodes — Chat Models, Memory, and supplyData()

How to build n8n **AI sub-nodes**: nodes that plug into the AI Agent (and other LangChain root nodes) as a model, memory, or other capability — as opposed to regular action nodes that an agent merely *calls* as tools.

## AI Sub-Node vs `usableAsTool`

Decide first which of these you're building:

| You want | Build |
|----------|-------|
| Your API's operations callable BY an AI agent (e.g. "create a contact" as an agent tool) | A normal action node with `usableAsTool: true` — no AI sub-node needed |
| Your LLM provider usable AS the agent's chat model | An AI sub-node with `outputs: [NodeConnectionTypes.AiLanguageModel]` |
| Your storage usable AS the agent's conversation memory | An AI sub-node with `outputs: [NodeConnectionTypes.AiMemory]` |

`usableAsTool: true` is a one-line flag on a regular node (and a lint requirement to declare either way). AI sub-nodes are a different paradigm: they implement `supplyData()` instead of `execute()`, have **no main inputs/outputs**, and connect via AI connection types.

## CLI Templates

Four of the seven `n8n-node new` templates scaffold AI sub-nodes:

```bash
n8n-node new n8n-nodes-myai --template programmatic/openai-chat-model
# or: programmatic/custom-chat-model | programmatic/custom-chat-model-example | programmatic/custom-chat-memory
```

| Template | What it scaffolds |
|----------|-------------------|
| `programmatic/openai-chat-model` | Chat model for an **OpenAI-compatible** API — `supplyModel(this, { type: 'openai', baseUrl, apiKey, model })`, no custom model class needed |
| `programmatic/custom-chat-model` | Chat model for a **custom protocol** — skeleton `BaseChatModel` subclass with `generate()`/`stream()` stubs |
| `programmatic/custom-chat-model-example` | Same as above but with a worked example implementation (good for learning the SDK shapes) |
| `programmatic/custom-chat-memory` | Conversation memory — `BaseChatHistory` storage + `WindowedChatMemory` + `supplyMemory()` |

## package.json Requirements

AI sub-node packages declare the AI Node SDK in **two coupled places** — the lint rule `@n8n/community-nodes/ai-node-package-json` enforces that both appear together:

```json
{
  "n8n": {
    "n8nNodesApiVersion": 1,
    "aiNodeSdkVersion": 1,
    "strict": true,
    "nodes": ["dist/nodes/MyChatModel/MyChatModel.node.js"],
    "credentials": ["dist/credentials/MyApi.credentials.js"]
  },
  "peerDependencies": {
    "n8n-workflow": "*",
    "@n8n/ai-node-sdk": "*"
  }
}
```

`@n8n/ai-node-sdk` is on the cloud-lint import allowlist, so AI sub-nodes remain eligible for n8n Cloud verification.

## Anatomy of a Chat Model Node

This is the actual shape the official template generates (note: `inputs: []`, a non-Main output, `supplyData` instead of `execute`, and an **inline `codex`** object placing the node in the AI section of the panel):

```typescript
import type { INodeType, INodeTypeDescription, ISupplyDataFunctions } from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';
import { supplyModel } from '@n8n/ai-node-sdk';

type ModelOptions = { temperature?: number };

export class MyChatModel implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Chat Model',
    name: 'myChatModel',
    icon: { light: 'file:../../icons/mymodel.svg', dark: 'file:../../icons/mymodel.dark.svg' },
    group: ['transform'],
    version: [1],
    description: 'Chat model node for the MyService API',
    defaults: { name: 'My Chat Model' },
    codex: {
      categories: ['assistant'],
      subcategories: {
        AI: ['Language Models', 'Root Nodes'],
        'Language Models': ['Chat Models (Recommended)'],
      },
      resources: { primaryDocumentation: [] },
    },
    inputs: [],
    outputs: [NodeConnectionTypes.AiLanguageModel],
    outputNames: ['Model'],
    credentials: [{ name: 'myServiceApi', required: true }],
    properties: [
      {
        displayName: 'Model',
        name: 'model',
        type: 'string',
        default: '',
        description: 'The model which will generate the completion',
      },
      {
        displayName: 'Options',
        name: 'options',
        placeholder: 'Add Option',
        description: 'Additional options to add',
        type: 'collection',
        default: {},
        options: [
          {
            displayName: 'Sampling Temperature',
            name: 'temperature',
            default: 0.7,
            typeOptions: { maxValue: 2, minValue: 0, numberPrecision: 1 },
            description: 'Controls randomness: lower is more deterministic',
            type: 'number',
          },
        ],
      },
    ],
  };

  async supplyData(this: ISupplyDataFunctions, itemIndex: number) {
    const credentials = await this.getCredentials('myServiceApi');
    const modelName = this.getNodeParameter('model', itemIndex) as string;
    const options = this.getNodeParameter('options', itemIndex, {}) as ModelOptions;

    return supplyModel(this, {
      type: 'openai',                       // OpenAI-compatible shortcut
      baseUrl: credentials.url as string,
      apiKey: credentials.apiKey as string,
      model: modelName,
      temperature: options.temperature,
    });
  }
}
```

Key differences from action nodes:

- **`supplyData(this: ISupplyDataFunctions, itemIndex: number): Promise<SupplyData>`** replaces `execute()`. `SupplyData` is `{ response: unknown, metadata?, closeFunction?, hints? }` — the SDK's `supplyModel`/`supplyMemory` build it for you.
- **`inputs: []`** and a single AI output with `outputNames`.
- The **inline `codex`** with `subcategories` controls placement in the AI section of the nodes panel (e.g. `AI → Language Models → Chat Models (Recommended)`).
- The template keeps shared icons in a project-level `icons/` directory and references them with relative paths (`file:../../icons/...`).

### AI Connection Types

From `NodeConnectionTypes` (n8n-workflow): `AiAgent`, `AiChain`, `AiDocument`, `AiEmbedding`, `AiLanguageModel`, `AiMemory`, `AiOutputParser`, `AiRetriever`, `AiReranker`, `AiTextSplitter`, `AiTool`, `AiVectorStore` — plus `Main` for regular nodes. Community AI sub-node templates currently target `AiLanguageModel` and `AiMemory`; the `@n8n/ai-node-sdk` ships `supplyModel` and `supplyMemory` helpers for exactly these.

## Custom (Non-OpenAI-Compatible) Chat Models

When the provider's protocol isn't OpenAI-compatible, subclass `BaseChatModel` and implement `generate()` (one-shot) and `stream()` (chunked):

```typescript
import type { IHttpRequestMethods } from 'n8n-workflow';
import {
  BaseChatModel,
  type ChatModelConfig,
  type GenerateResult,
  type Message,
  type StreamChunk,
} from '@n8n/ai-node-sdk';

interface ModelConfig extends ChatModelConfig {
  url: string;
}

export class CustomChatModel extends BaseChatModel<ModelConfig> {
  constructor(modelId: string, /* injected request helpers */, config: ModelConfig) {
    super('custom-provider', modelId, config);
  }

  async generate(messages: Message[], config?: ModelConfig): Promise<GenerateResult> {
    const merged = this.mergeConfig(config);
    // 1. Map n8n Message[] ({ role, content: [{ type: 'text', text }] }) to the provider format
    // 2. Call the API (inject this.helpers.httpRequestWithAuthentication.call from supplyData)
    // 3. Map the response back:
    return {
      finishReason: 'stop',
      usage: { promptTokens: 0, completionTokens: 0, totalTokens: 0 },
      message: { role: 'assistant', content: [{ type: 'text', text: '...' }] },
    };
  }

  async *stream(messages: Message[], config?: ModelConfig): AsyncIterable<StreamChunk> {
    // Open a stream against the API, then:
    // yield { type: 'text-delta', delta: chunk } for each chunk
    yield { type: 'finish', finishReason: 'stop' };
  }
}
```

Then in `supplyData`: `return supplyModel(this, new CustomChatModel(modelName, requests, { url, temperature }))`.

Pattern note from the official template: HTTP calls are injected into the model class as callback functions created inside `supplyData` (where `this.helpers.httpRequestWithAuthentication.call(this, ...)` is available) — the model class itself stays transport-agnostic. The SDK also exports `parseSSEStream` for server-sent-event streaming APIs.

## Chat Memory Nodes

Memory nodes store conversation history. The template wires three pieces: a `BaseChatHistory` implementation (your storage), `WindowedChatMemory` (context-window limiting), and `supplyMemory`:

```typescript
import type { INodeType, INodeTypeDescription, ISupplyDataFunctions } from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';
import { supplyMemory, WindowedChatMemory } from '@n8n/ai-node-sdk';
import { MyChatHistory } from './memory';   // extends BaseChatHistory

export class MyChatMemory implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Memory',
    name: 'myChatMemory',
    icon: { light: 'file:mymemory.svg', dark: 'file:mymemory.dark.svg' },
    group: ['transform'],
    version: [1],
    description: 'Store conversation history in MyService',
    defaults: { name: 'My Memory' },
    codex: {
      categories: ['assistant'],
      subcategories: { AI: ['Memory', 'Root Nodes'], Memory: ['Other memories'] },
      resources: { primaryDocumentation: [] },
    },
    inputs: [],
    outputs: [NodeConnectionTypes.AiMemory],
    outputNames: ['Memory'],
    credentials: [],
    properties: [
      {
        displayName: 'Session ID',
        name: 'sessionId',
        type: 'string',
        default: '={{ $json.sessionId }}',
        description: 'Unique identifier for the conversation session',
        placeholder: 'e.g. user-123',
      },
      {
        displayName: 'Options',
        name: 'options',
        placeholder: 'Add Option',
        description: 'Additional options for memory management',
        type: 'collection',
        default: {},
        options: [
          {
            displayName: 'Window Size',
            name: 'windowSize',
            type: 'number',
            default: 10,
            description: 'Number of recent message pairs to keep in context',
            typeOptions: { minValue: 1, maxValue: 100 },
          },
        ],
      },
    ],
  };

  async supplyData(this: ISupplyDataFunctions, itemIndex: number) {
    const sessionId = this.getNodeParameter('sessionId', itemIndex) as string;
    const options = this.getNodeParameter('options', itemIndex, {}) as { windowSize?: number };

    const history = new MyChatHistory(sessionId);
    const memory = new WindowedChatMemory(history, { windowSize: options.windowSize ?? 10 });
    return supplyMemory(this, memory);
  }
}
```

## SDK Surface (`@n8n/ai-node-sdk`)

| Export | Purpose |
|--------|---------|
| `supplyModel(ctx, modelOrOptions)` | Build the `SupplyData` for a chat model — accepts `{ type: 'openai', baseUrl, apiKey, model, ... }` for OpenAI-compatible APIs or a `BaseChatModel` instance |
| `supplyMemory(ctx, memory)` | Build the `SupplyData` for a memory node |
| `BaseChatModel` | Abstract base for custom models — implement `generate()` and `stream()` |
| `BaseChatHistory` / `BaseChatMemory` / `WindowedChatMemory` | Memory building blocks |
| `parseSSEStream` | Helper for server-sent-event streaming responses |
| Types: `Message`, `GenerateResult`, `StreamChunk`, `TokenUsage`, `FinishReason`, `ChatModelConfig`, `Tool`, `ToolCall` | Provider-agnostic message/result shapes |

## Validation

The standard gate protocol (`references/validation.md`) applies unchanged — same lint, build, runtime, and release gates. AI-specific notes:

- `ai-node-package-json` (lint error): `n8n.aiNodeSdkVersion` and the `@n8n/ai-node-sdk` peer dependency must be declared together.
- `@n8n/ai-node-sdk` imports pass the cloud-only `no-restricted-imports` rule.
- Runtime smoke test (Gate 4): add an **AI Agent** node in the dev editor, connect your sub-node to its Model/Memory port, and run a chat turn — that exercises `supplyData` end-to-end.
- `usableAsTool` must still be declared on the class (lint rule applies to every node); for sub-nodes set it to `false` — they are wired by connection, not invoked as tools.
