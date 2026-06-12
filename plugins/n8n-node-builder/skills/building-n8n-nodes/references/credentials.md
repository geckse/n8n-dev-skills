# Credentials Reference

Complete patterns for all n8n credential types.

## Table of Contents

1. [Credential File Structure](#credential-file-structure)
2. [API Key Authentication](#api-key-authentication)
3. [Bearer Token Authentication](#bearer-token-authentication)
4. [Basic Auth Authentication](#basic-auth-authentication)
5. [OAuth2 Authentication](#oauth2-authentication)
6. [Custom Authentication](#custom-authentication)
7. [preAuthentication (Session-Token APIs)](#preauthentication-session-token-apis)
8. [Credential Testing](#credential-testing)
9. [Multiple Auth Methods](#multiple-auth-methods)
10. [Lint-Enforced Naming](#lint-enforced-naming)
11. [Exposing Credentials to the HTTP Request Node](#exposing-credentials-to-the-http-request-node)

## Credential File Structure

Every credential file lives in `credentials/<Name>Api.credentials.ts` and exports a class implementing `ICredentialType`.

The linter enforces credential naming: the `name` field must start with a lowercase letter and end with `Api` (e.g. `myServiceApi`); the class name must also end with `Api`; OAuth2 credentials must use the `OAuth2Api` class-name suffix and include 'OAuth2' in both `name` and `displayName`. All are error-level rules (`cred-class-name-field-conventions`, `cred-class-name-suffix`, `cred-class-oauth2-naming`). See [Lint-Enforced Naming](#lint-enforced-naming) below.

```typescript
import type {
  IAuthenticateGeneric,
  ICredentialTestRequest,
  ICredentialType,
  INodeProperties,
  Icon,
} from 'n8n-workflow';

export class MyServiceApi implements ICredentialType {
  // Internal name — must match the node's credentials[].name
  // (lint-enforced: starts lowercase, ends with 'Api')
  name = 'myServiceApi';

  // Display name shown in the credentials UI
  displayName = 'My Service API';

  // URL to the service's API docs (shown as help link)
  documentationUrl = 'https://docs.myservice.com/api';

  // Icon shown in the credentials list (REQUIRED by linter)
  // Place a copy of your SVG icon in the credentials/ folder
  icon: Icon = 'file:myService.svg';

  // User-facing input fields
  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];

  // How n8n injects credentials into requests
  authenticate: IAuthenticateGeneric = {
    type: 'generic',
    properties: {
      headers: {
        Authorization: '=Bearer {{$credentials.apiKey}}',
      },
    },
  };

  // Lightweight test request to validate credentials
  test: ICredentialTestRequest = {
    request: {
      baseURL: 'https://api.myservice.com/v1',
      url: '/me',
    },
  };
}
```

## API Key Authentication

### Via Header

```typescript
properties: INodeProperties[] = [
  {
    displayName: 'API Key',
    name: 'apiKey',
    type: 'string',
    typeOptions: { password: true },
    default: '',
  },
];

authenticate: IAuthenticateGeneric = {
  type: 'generic',
  properties: {
    headers: {
      'X-API-Key': '={{$credentials.apiKey}}',
    },
  },
};
```

### Via Query String

```typescript
authenticate: IAuthenticateGeneric = {
  type: 'generic',
  properties: {
    qs: {
      api_key: '={{$credentials.apiKey}}',
    },
  },
};
```

### Via Request Body

```typescript
authenticate: IAuthenticateGeneric = {
  type: 'generic',
  properties: {
    body: {
      apiKey: '={{$credentials.apiKey}}',
    },
  },
};
```

## Bearer Token Authentication

```typescript
properties: INodeProperties[] = [
  {
    displayName: 'Access Token',
    name: 'accessToken',
    type: 'string',
    typeOptions: { password: true },
    default: '',
  },
];

authenticate: IAuthenticateGeneric = {
  type: 'generic',
  properties: {
    headers: {
      Authorization: '=Bearer {{$credentials.accessToken}}',
    },
  },
};
```

## Basic Auth Authentication

```typescript
properties: INodeProperties[] = [
  {
    displayName: 'Username',
    name: 'username',
    type: 'string',
    default: '',
  },
  {
    displayName: 'Password',
    name: 'password',
    type: 'string',
    typeOptions: { password: true },
    default: '',
  },
];

authenticate: IAuthenticateGeneric = {
  type: 'generic',
  properties: {
    auth: {
      username: '={{$credentials.username}}',
      password: '={{$credentials.password}}',
    },
  },
};
```

## OAuth2 Authentication

```typescript
import type {
  ICredentialType,
  INodeProperties,
  Icon,
} from 'n8n-workflow';

export class MyServiceOAuth2Api implements ICredentialType {
  name = 'myServiceOAuth2Api';
  displayName = 'My Service OAuth2 API';
  icon: Icon = 'file:myService.svg';

  // Extend the built-in OAuth2 credential type
  extends = ['oAuth2Api'];

  documentationUrl = 'https://docs.myservice.com/oauth2';

  properties: INodeProperties[] = [
    {
      displayName: 'Grant Type',
      name: 'grantType',
      type: 'hidden',
      default: 'authorizationCode',
    },
    {
      displayName: 'Authorization URL',
      name: 'authUrl',
      type: 'hidden',
      default: 'https://myservice.com/oauth/authorize',
    },
    {
      displayName: 'Access Token URL',
      name: 'accessTokenUrl',
      type: 'hidden',
      default: 'https://myservice.com/oauth/token',
    },
    {
      displayName: 'Scope',
      name: 'scope',
      type: 'hidden',
      default: 'read write',
    },
    {
      displayName: 'Auth URI Query Parameters',
      name: 'authQueryParameters',
      type: 'hidden',
      default: '',
    },
    {
      displayName: 'Authentication',
      name: 'authentication',
      type: 'hidden',
      default: 'header',
    },
  ];
}
```

In the node's credential reference:
```typescript
credentials: [
  {
    name: 'myServiceOAuth2Api',
    required: true,
  },
],
```

## Custom Authentication

For non-standard auth flows where you need full control:

```typescript
import type {
  ICredentialType,
  INodeProperties,
  ICredentialDataDecryptedObject,
  IHttpRequestOptions,
  Icon,
} from 'n8n-workflow';

export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'My Service API';
  icon: Icon = 'file:myService.svg';

  properties: INodeProperties[] = [
    {
      displayName: 'Domain',
      name: 'domain',
      type: 'string',
      default: '',
      placeholder: 'e.g. https://yourcompany.myservice.com',
    },
    {
      displayName: 'API Token',
      name: 'apiToken',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];

  // Custom authenticate method
  async authenticate(
    credentials: ICredentialDataDecryptedObject,
    requestOptions: IHttpRequestOptions,
  ): Promise<IHttpRequestOptions> {
    requestOptions.headers = requestOptions.headers || {};
    requestOptions.headers['Authorization'] = `Token ${credentials.apiToken}`;
    requestOptions.headers['X-Tenant'] = credentials.domain as string;
    return requestOptions;
  }

  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials?.domain}}',
      url: '/api/v1/verify',
    },
  };
}
```

## preAuthentication (Session-Token APIs)

For APIs that exchange long-lived credentials (e.g. an API key or username/password) for a short-lived session token, implement `preAuthentication`. n8n calls it only when the expirable property is empty or expired, stores the returned value in the credential data, and re-runs it automatically when the token expires.

The exact signature on `ICredentialType`:

```typescript
preAuthentication?: (
  this: IHttpRequestHelper,
  credentials: ICredentialDataDecryptedObject,
) => Promise<IDataObject>;
```

Pair it with a hidden property carrying `typeOptions: { expirable: true }` that stores the session token (this is the pattern used by the built-in Metabase credential):

```typescript
import type {
  IAuthenticateGeneric,
  ICredentialDataDecryptedObject,
  ICredentialTestRequest,
  ICredentialType,
  IHttpRequestHelper,
  INodeProperties,
  Icon,
} from 'n8n-workflow';

export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'My Service API';
  documentationUrl = 'https://docs.myservice.com/api';
  icon: Icon = 'file:myService.svg';

  properties: INodeProperties[] = [
    // Hidden, expirable property holding the session token
    {
      displayName: 'Session Token',
      name: 'sessionToken',
      type: 'hidden',
      typeOptions: {
        expirable: true,
      },
      default: '',
    },
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];

  // Only called when "sessionToken" (the expirable property) is empty or expired
  async preAuthentication(this: IHttpRequestHelper, credentials: ICredentialDataDecryptedObject) {
    const { token } = (await this.helpers.httpRequest({
      method: 'POST',
      url: 'https://api.myservice.com/v1/session',
      body: { apiKey: credentials.apiKey },
    })) as { token: string };
    // Returned object is merged into the stored credential data
    return { sessionToken: token };
  }

  authenticate: IAuthenticateGeneric = {
    type: 'generic',
    properties: {
      headers: {
        'X-Session-Token': '={{$credentials.sessionToken}}',
      },
    },
  };

  test: ICredentialTestRequest = {
    request: {
      baseURL: 'https://api.myservice.com/v1',
      url: '/me',
    },
  };
}
```

## Credential Testing

The `test` property sends a lightweight request to verify credentials work. It runs automatically when the user saves the credential in the credentials dialog; users can re-run it via the 'Retry credential test' action.

```typescript
// Simple test against a known endpoint:
test: ICredentialTestRequest = {
  request: {
    baseURL: 'https://api.myservice.com/v1',
    url: '/me',
  },
};

// Test with dynamic base URL from credentials:
test: ICredentialTestRequest = {
  request: {
    baseURL: '={{$credentials?.domain}}',
    url: '/api/v1/ping',
  },
};
```

By default a 2xx response passes and any error response fails. You can customize this with the optional `rules` array on `ICredentialTestRequest`: `{ type: 'responseCode', properties: { value: 403, message: '...' } }` to fail on a specific status code, or `{ type: 'responseSuccessBody', properties: { key, value, message } }` to fail when an API returns 200 with an error body.

```typescript
// For APIs that return 200 with an error payload (pattern from the built-in Slack credential):
test: ICredentialTestRequest = {
  request: {
    baseURL: 'https://api.myservice.com/v1',
    url: '/auth/test',
  },
  rules: [
    {
      type: 'responseSuccessBody',
      properties: {
        key: 'error',
        value: 'invalid_auth',
        message: 'Invalid access token',
      },
    },
  ],
};

// Fail on a specific status code with a custom message:
test: ICredentialTestRequest = {
  request: {
    baseURL: 'https://api.myservice.com/v1',
    url: '/me',
  },
  rules: [
    {
      type: 'responseCode',
      properties: {
        value: 403,
        message: 'Does your API key have the required scopes?',
      },
    },
  ],
};
```

## Multiple Auth Methods

A node can support multiple credential types. Users choose which one to use:

```typescript
// In the node file:
credentials: [
  {
    name: 'myServiceApi',
    required: true,
    displayOptions: {
      show: {
        authentication: ['apiKey'],
      },
    },
  },
  {
    name: 'myServiceOAuth2Api',
    required: true,
    displayOptions: {
      show: {
        authentication: ['oAuth2'],
      },
    },
  },
],
properties: [
  {
    displayName: 'Authentication',
    name: 'authentication',
    type: 'options',
    options: [
      { name: 'API Key', value: 'apiKey' },
      { name: 'OAuth2', value: 'oAuth2' },
    ],
    default: 'apiKey',
  },
  // ... rest of properties
],
```

## Lint-Enforced Naming

The community-nodes linter (`@n8n/eslint-plugin-community-nodes`, run via `npm run lint`) enforces credential naming and testing conventions at error level:

| Rule | Requirement |
|------|-------------|
| `cred-class-name-field-conventions` | The `name` field must start with a lowercase letter and end with `Api` (e.g. `myServiceApi`) |
| `cred-class-name-suffix` | The class name must end with `Api` (e.g. `MyServiceApi`) |
| `cred-class-oauth2-naming` | OAuth2 credentials: class name must end with `OAuth2Api`, and both `name` and `displayName` must include 'OAuth2' |
| `credential-documentation-url` | `documentationUrl` must be a valid URL (or a lowercase alphanumeric slug, where slugs are allowed) |
| `credential-test-required` | Every credential needs a `test` property or a node-side `testedBy` — except classes extending `oAuth2Api`, which are exempt (do NOT add `test` to them; they are validated via the OAuth connect flow) |

See [validation.md](validation.md) for the full lint rule catalog and the validation gate protocol.

## Exposing Credentials to the HTTP Request Node

Add `httpRequestNode` to your credential class to make it selectable as a predefined credential type in the generic HTTP Request node:

```typescript
httpRequestNode = {
  // Service name shown in the HTTP Request node's credential dropdown
  name: 'My Service',
  // Help link to the service's API docs
  docsUrl: 'https://docs.myservice.com/api',
  // Pre-filled API base URL
  apiBaseUrl: 'https://api.myservice.com/v1/',
};
```

Notes:
- `apiBaseUrl` and `apiBaseUrlPlaceholder` are mutually exclusive — use `apiBaseUrlPlaceholder` when the base URL varies per instance (e.g. self-hosted services).
- Related (optional) `ICredentialType` fields: `supportedNodes?: string[]` lists the node names that use this credential, and `restrictToSupportedNodes?: true` makes the execution engine refuse to decrypt the credential for any node not in `supportedNodes` — including the HTTP Request node and its tool variants.

## Package.json Registration

Credentials must be registered in `package.json` under the `n8n` attribute:

```json
{
  "n8n": {
    "n8nNodesApiVersion": 1,
    "strict": true,
    "credentials": [
      "dist/credentials/MyServiceApi.credentials.js"
    ],
    "nodes": [
      "dist/nodes/MyService/MyService.node.js"
    ]
  }
}
```

Paths point to the compiled JavaScript files in `dist/`, not the TypeScript source.

## Domain/URL Credentials

When the API base URL varies per customer (self-hosted services):

```typescript
properties: INodeProperties[] = [
  {
    displayName: 'Domain',
    name: 'domain',
    type: 'string',
    default: 'https://myinstance.myservice.com',
    placeholder: 'e.g. https://your-instance.myservice.com',
  },
  {
    displayName: 'API Key',
    name: 'apiKey',
    type: 'string',
    typeOptions: { password: true },
    default: '',
  },
];
test: ICredentialTestRequest = {
  request: {
    baseURL: '={{$credentials.domain}}',
    url: '/api/v1/me',
  },
};
```

## Custom Credential Test (testedBy)

For complex validation not suited to a simple HTTP request, use `testedBy` instead of the `test` property:

```typescript
// In credential file: (no test property — use testedBy instead)

// In node file, reference it:
credentials: [
  { name: 'myServiceApi', required: true, testedBy: 'myServiceApiTest' },
],

// Also in node file, add test method:
methods = {
  credentialTest: {
    async myServiceApiTest(
      this: ICredentialTestFunctions,
      credential: ICredentialsDecrypted,
    ): Promise<INodeCredentialTestResult> {
      try {
        // Custom validation logic
        return { status: 'OK', message: 'Connection successful' };
      } catch (error) {
        return { status: 'Error', message: error.message };
      }
    },
  },
};
```

## Injection Locations

The `authenticate.properties` object supports these locations:

| Location | Where it's injected | Example |
|----------|-------------------|---------|
| `headers` | HTTP headers | `Authorization: Bearer ...` |
| `qs` | URL query parameters | `?api_key=...` |
| `body` | Request body | `{ "token": "..." }` |
| `auth` | Basic auth (username/password) | `{ username, password }` |

**Expression syntax:** Always use `'={{$credentials.fieldName}}'` to reference credential values. Note the plural `$credentials` — using singular `$credential` is a common mistake.

## Common Credential Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `icon` property on credential | Add `icon: Icon = 'file:myservice.svg'` and place SVG in credentials/ |
| Missing `typeOptions: { password: true }` on secrets | Always mask API keys, tokens, passwords |
| Wrong expression: `$credential.apiKey` | Use `$credentials.apiKey` (plural) |
| Forgot to list credential in `package.json` | Add to `n8n.credentials` array |
| Test endpoint requires auth but `authenticate` not set | `test.request` auto-uses the `authenticate` config |
| OAuth2 showing editable URL fields | Use `type: 'hidden'` for auth/token URLs |

## Best Practices

- Always use `typeOptions: { password: true }` for secret fields (API keys, tokens, passwords)
- Add `icon` property with `Icon` type to credential classes (required by linter)
- Include a `documentationUrl` pointing to the service's auth documentation
- Always implement a `test` request or use `testedBy` — except for OAuth2 credentials that extend `oAuth2Api`, which are validated through the OAuth connection flow and are exempt from the `credential-test-required` lint rule (do not add a `test` property to them)
- For OAuth2, extend `oAuth2Api` and set authorization/token URLs as hidden fields
- Place a copy of your SVG icon in the `credentials/` folder and reference as `'file:name.svg'`
