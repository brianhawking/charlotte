# Charlotte — Backend Build Guide

Charlotte is an internal iOS developer tool that acts as an intelligent mock manager. It sits between the iOS app and the real backend APIs, intercepting traffic and optionally serving mock responses instead. It has a web UI (the HTML files in `public/`) and a Node.js server (`server.js`) that does all the heavy lifting.

This document explains everything the backend needs to do. Build `server.js` based on this spec.

---

## Project Structure

```
charlotte/
├── public/               # Static UI files (already built)
│   ├── live-traffic.html
│   ├── mock-library.html
│   ├── templates.html
│   └── scenarios.html
├── mocks/                # Mock response files (local only, not committed)
│   └── {endpoint-folder}/
│       └── {mock-name}.json
├── templates/            # Scenario template definitions (committed to repo)
│   └── {template-name}.json
├── scenarios/            # Saved scenario bundles (local only, not committed)
│   └── {scenario-name}.json
├── server.js             # YOU ARE BUILDING THIS
├── package.json
└── CLAUDE.md             # This file
```

---

## Core Concepts

### Mock
A single saved JSON response for one API endpoint. Stored as a file in `mocks/{endpoint-folder}/{mock-name}.json`.

A mock file looks like this:
```json
{
  "_meta": {
    "name": "Login — standard",
    "endpoint": "/login",
    "method": "GET",
    "variantKey": "isPreAuth-false",
    "createdAt": "2026-05-20T10:00:00Z"
  },
  "_request": {
    "body": { "isPreAuth": false },
    "params": {}
  },
  "_headers": {
    "auth-token": "mock-auth-token-offline",
    "transaction-info": "mock-transaction-info"
  },
  "profileReferenceId": "dyEEnaW0hmgtys4cOn",
  "customerReferenceId": "VokhBI7LYJ9zM+4RFEOkqQ==",
  "loginStatus": "SUCCESS",
  "mobileOnly": false
}
```

Key fields:
- `_meta` — Charlotte metadata, not sent to the app
- `_request` — the request body/params that identify this variant (used for routing in offline mode)
- `_headers` — response headers to inject (used for auth token injection)
- Everything else is the actual response body sent to the app

### Endpoint Folder Naming
Convert the API path to a folder name by replacing `/` with `-` and removing the leading `-`.

Examples:
- `/login` → `login`
- `/private/1364896/loyalty-benefits/retrieve-account-benefits` → `private-1364896-loyalty-benefits-retrieve-account-benefits`
- `/customers/~/messaging-preferences` → `customers-~-messaging-preferences`

Numbers in paths are meaningful service IDs — keep them.

### Template
A developer-curated list of API endpoints that belong together for a feature. No response content — just the API list. Templates are committed to the repo.

```json
{
  "name": "Rewards Balance",
  "description": "The rewards hub showing balance, earn activity, and redemption options",
  "source": "plugin-map",
  "createdAt": "2026-05-20T10:00:00Z",
  "apis": [
    { "method": "POST", "path": "/protected/375751/loyalty/accounts/digital-account-view/get-accounts" },
    { "method": "GET",  "path": "/loyalty/accounts/transaction-history/transactions" },
    { "method": "GET",  "path": "/private/1364896/loyalty-benefits/retrieve-account-benefits" }
  ]
}
```

Source can be: `"plugin-map"`, `"captured"`, or `"manual"`.

### Scenario
A named bundle of mock responses for a specific user state. Created from a template by Claude. Local only, not committed.

```json
{
  "name": "3 Branded — Miles and Cash",
  "origin": "generated",
  "templateName": "Rewards Balance",
  "createdAt": "2026-05-20T14:00:00Z",
  "active": false,
  "mocks": {
    "/login": "login/login-standard.json",
    "/protected/375751/loyalty/accounts/digital-account-view/get-accounts": "protected-375751-loyalty-accounts-digital-account-view-get-accounts/miles-and-cash.json"
  }
}
```

The `mocks` map points to relative file paths inside the `mocks/` folder.

### Baseline APIs
Some APIs are always needed regardless of feature (login, auth, accounts). These are not part of any template but are always served in offline mode. Define them as a hardcoded list in the server:

```js
const BASELINE_ENDPOINTS = [
  '/login',
  '/private/1595110/edge/customers/~/accounts',
  '/private/1612081/v1/force-upgrade',
  '/identity/enhanced-authentication/devices'
];
```

---

## Server Modes

### Passthrough Mode (default)
All traffic goes to the real API. Charlotte intercepts and logs every request/response for the Live Traffic view. No mocks are served.

### Offline Mode
All traffic is intercepted. Charlotte looks up the appropriate mock file and returns it. If no mock exists for an endpoint, return a clear error:
```json
{
  "error": "No mock found",
  "endpoint": "/path/that/was/called",
  "message": "Charlotte is in offline mode but no mock exists for this endpoint."
}
```

Mode is stored in `state.json` and toggled via the UI.

---

## state.json

Persists server state across restarts:
```json
{
  "mode": "passthrough",
  "activeScenario": null,
  "serving": {
    "/login": "login/login-standard.json",
    "/protected/375751/loyalty/accounts/digital-account-view/get-accounts": "protected-375751-loyalty-accounts-digital-account-view-get-accounts/miles-and-cash.json"
  },
  "focused": []
}
```

- `mode` — `"passthrough"` or `"offline"`
- `activeScenario` — name of active scenario or null
- `serving` — map of endpoint path → mock file path currently being served
- `focused` — list of request objects the developer has focused in Live Traffic

---

## Mock Routing (Offline Mode)

When a request comes in offline:

1. Convert request path to folder name
2. Look up `state.serving[path]` — if found, use that mock file
3. If not found, check if any mock exists in the folder — if only one, use it
4. If multiple mocks exist and none is designated, return the no-mock error
5. Read the mock file, strip `_meta` and `_request` fields, inject `_headers` as response headers, return the body

### Auth Token Injection
If a mock file contains `_headers`, inject those as actual HTTP response headers. This is how the login mock provides a fake auth token that downstream calls will see.

---

## Variant Naming (Claude Call)

When a mock is saved and a collision is detected (a mock already exists for that endpoint folder), call Claude to generate a short unique label from the request body.

```js
async function getVariantLabel(requestBody, endpointPath) {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 50,
    messages: [{
      role: 'user',
      content: `Given this API endpoint: ${endpointPath}
And this request body: ${JSON.stringify(requestBody)}

Return a single short label (2-4 words, hyphenated, lowercase) that uniquely identifies what makes this request distinct from other calls to the same endpoint. Return only the label, nothing else.

Examples:
- "card-onboarding-states"
- "is-pre-auth-false"
- "enterprise-marquee-plugin"`
    }]
  });
  return response.content[0].text.trim();
}
```

The label becomes part of the filename: `{endpoint-folder}/{label}.json`

---

## API Endpoints the UI Needs

Build these Express routes:

### State
- `GET /api/state` — return current state.json
- `POST /api/state` — update state (mode, activeScenario, serving map)

### Live Traffic
- `GET /api/traffic` — return in-memory traffic log (last 500 requests)
- `POST /api/traffic/clear` — clear the log
- `POST /api/traffic/focus` — add a request to the focused list
- `DELETE /api/traffic/focus/:id` — remove from focused list
- `POST /api/traffic/save-mock` — save a focused request as a mock (triggers variant naming if needed)
- `POST /api/traffic/save-template` — save focused endpoints as a new template (no content, just the API list)
- `POST /api/traffic/save-scenario` — save focused requests as a captured scenario (with content)

### Mocks
- `GET /api/mocks` — return all mocks grouped by endpoint folder
- `GET /api/mocks/:folder/:filename` — return a single mock file
- `PUT /api/mocks/:folder/:filename` — update a mock file
- `DELETE /api/mocks/:folder/:filename` — delete a mock file
- `POST /api/mocks/:folder/:filename/serve` — set as the serving mock for that endpoint
- `DELETE /api/mocks/:folder/serve` — stop serving any mock for that endpoint
- `POST /api/mocks/generate` — generate a single mock via Claude (see Claude Integration)

### Templates
- `GET /api/templates` — return all templates
- `POST /api/templates` — create a new template
- `PUT /api/templates/:name` — update a template (add/remove endpoints)
- `DELETE /api/templates/:name` — delete a template

### Scenarios
- `GET /api/scenarios` — return all scenarios
- `POST /api/scenarios/:name/activate` — activate a scenario (updates serving map in state)
- `POST /api/scenarios/:name/deactivate` — deactivate
- `DELETE /api/scenarios/:name` — delete
- `PUT /api/scenarios/:name/mocks/:endpoint` — update which mock serves a given endpoint within a scenario
- `POST /api/scenarios/generate` — generate a full scenario suite from a template via Claude (see Claude Integration)

### Endpoints
- `GET /api/endpoints` — return list of all known endpoints (from mocks folder, used for dropdowns in UI)

---

## Claude Integration

### 1. Generate Single Mock
`POST /api/mocks/generate`

Request body:
```json
{
  "endpoint": "/protected/375751/loyalty/accounts/digital-account-view/get-accounts",
  "method": "POST",
  "name": "Miles and Cash — zero balance",
  "description": "User with Miles and Cash rewards, zero balance on both currencies, no active partner offers"
}
```

Claude prompt approach:
- Tell Claude the endpoint path, method, and user description
- If a feature map exists for this endpoint, include the response field schema as context
- Ask Claude to return only valid JSON matching the expected response structure
- Save the result as a mock file

### 2. Generate Variant Label
Called automatically when saving a mock that collides with an existing endpoint folder. See Variant Naming section above.

### 3. Generate Variants from Existing Mock
`POST /api/mocks/:folder/:filename/variants`

Takes an existing mock response and asks Claude to generate meaningful real-world variations of it. For example, given a rewards balance response, Claude generates:
- Zero balance variant
- Miles only variant  
- Ineligible user variant

Claude should understand the domain from the response shape and generate 3-5 named variants. Each gets saved as a separate mock file in the same folder.

### 4. Generate Full Scenario Suite
`POST /api/scenarios/generate`

Request body:
```json
{
  "templateName": "Rewards Balance",
  "scenarioName": "Venture X — Gold tier",
  "userDescription": "Venture X cardholder, Gold loyalty tier, $15k balance, eligible for all 5 partner offers"
}
```

This is the most important Claude call. Claude must:
1. Read the template to get the list of APIs
2. Generate ALL responses at once in a single call — not one by one
3. Ensure consistency across all responses — the same accountReferenceId, productId, balances, and user details must appear correctly in every response that references them
4. Return a JSON object mapping each endpoint to its complete response body

Prompt approach:
- Pass the full template API list with any available schema hints
- Pass the user description
- Ask Claude to return a single JSON object where keys are endpoint paths and values are the response bodies
- Stress that IDs, account references, and user state must be consistent across all responses
- Claude should generate baseline API responses too (login, accounts) unless told not to

Save each response as a mock file and create the scenario JSON pointing to them.

---

## Traffic Logging

Keep the last 500 requests in memory (not on disk). Each entry:
```js
{
  id: uuid(),
  timestamp: Date.now(),
  method: 'POST',
  path: '/internal-ops/feature-experimentation/servicing/decide-variation',
  statusCode: 200,
  requestBody: { ... },
  requestHeaders: { ... },
  responseBody: { ... },
  responseHeaders: { ... },
  size: 701,
  source: 'live',        // 'live' or 'mock'
  variantKey: 'Card_OnboardingStates_Plugin',  // extracted from request, null if not applicable
  focused: false
}
```

The `variantKey` is whatever Claude or the server determines makes this request distinct — shown as the subtle secondary label in the Live Traffic view.

---

## Proxy Setup

Charlotte acts as an HTTP proxy. The iOS app should be configured (via Charles or similar) to route through Charlotte's local port.

When in passthrough mode:
1. Receive request from iOS app
2. Log it to the traffic buffer
3. Forward to the real API
4. Log the response
5. Return response to the app

When in offline mode:
1. Receive request
2. Log it
3. Look up mock
4. Return mock response (with injected headers)

---

## Key Technical Notes

- Use Express for the HTTP server
- The proxy functionality can use `http-proxy` or `axios` to forward requests
- Store `state.json` in the project root, read on startup, write on every state change
- Mock files are JSON — always validate before saving, return `{ valid: false, error: "..." }` if invalid
- The `_meta`, `_request`, and `_headers` fields are Charlotte internals — strip them before sending to the iOS app
- For Claude API calls, use `@anthropic-ai/sdk`
- The server should run on port 3001 by default (configurable)
- The UI is served as static files from the `public/` folder at the root route

---

## What We Are NOT Building Yet

- Authentication / multi-user support
- Android support (iOS only for now)
- Automated UI testing / validation loop
- Plugin map generation from codebase (that's a separate CLI tool)
- Git integration for committing templates

---

## Summary of File Conventions

| What | Where | Committed? |
|------|-------|-----------|
| UI HTML files | `public/` | Yes |
| Mock files | `mocks/{folder}/{name}.json` | No |
| Template files | `templates/{name}.json` | Yes |
| Scenario files | `scenarios/{name}.json` | No |
| Server state | `state.json` | No |
| Server code | `server.js` | Yes |
