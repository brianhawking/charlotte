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

All Claude calls in server.js are made via `claude -p` — the Claude Code CLI running as a subprocess. There is no Anthropic API key or SDK involved. Claude Code must be installed and authenticated on the developer's machine for these calls to work.

### How to call Claude from server.js

```js
const { execSync } = require('child_process');

function callClaude(prompt) {
  try {
    const escaped = prompt.replace(/"/g, '\"').replace(/`/g, '\`');
    const result = execSync(`claude -p "${escaped}"`, {
      encoding: 'utf8',
      timeout: 120000  // 2 min timeout for longer generation tasks
    });
    return result.trim();
  } catch (err) {
    throw new Error(`Claude call failed: ${err.message}`);
  }
}
```

For calls that need to return JSON, instruct Claude in the prompt to return only valid JSON with no markdown, no backticks, no preamble.

---

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

Example prompt:
```
Generate a realistic JSON response for this iOS API endpoint:
Endpoint: POST /protected/375751/loyalty/accounts/digital-account-view/get-accounts
User description: User with Miles and Cash rewards, zero balance on both currencies, no active partner offers

Return only valid JSON. No markdown, no backticks, no explanation.
```

Parse the result as JSON and save as a mock file.

### 2. Generate Variant Label
Called automatically when saving a mock that collides with an existing endpoint folder. See Variant Naming section above.

Example prompt:
```
Given this API endpoint: /internal-ops/feature-experimentation/servicing/decide-variation
And this request body: {"featureKey": "Card_OnboardingStates_Plugin"}

Return a single short label (2-4 words, hyphenated, lowercase) that uniquely identifies what makes this request distinct from other calls to the same endpoint. Return only the label, nothing else.
```

### 3. Generate Variants from Existing Mock
`POST /api/mocks/:folder/:filename/variants`

Takes an existing mock response and asks Claude to generate meaningful real-world variations. For example, given a rewards balance response, Claude generates zero balance, miles only, ineligible user variants.

Example prompt:
```
Here is an existing API mock response for endpoint: /protected/375751/loyalty/accounts/digital-account-view/get-accounts

{...existing response JSON...}

Generate 3-5 meaningful real-world variations of this response. Each variation should represent a different realistic user state (e.g. zero balance, different currency types, ineligible user, etc).

Return only a JSON array where each item has:
- "name": short descriptive name (e.g. "zero-balance")  
- "response": the full response JSON for that variant

No markdown, no backticks, no explanation.
```

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

This is the most important Claude call. Claude must generate ALL responses at once — not one by one — and ensure consistency across all responses (same accountReferenceId, productId, balances, and user details throughout).

Example prompt:
```
Generate a complete suite of mock API responses for an iOS app scenario.

User description: Venture X cardholder, Gold loyalty tier, $15k balance, eligible for all 5 partner offers

APIs to generate responses for:
- POST /protected/375751/loyalty/accounts/digital-account-view/get-accounts
- GET /loyalty/accounts/transaction-history/transactions
- GET /private/1364896/loyalty-benefits/retrieve-account-benefits
- GET /private/1364896/loyalty-benefits/retrieve-customer-benefits
- (etc — full list from template)

Critical requirements:
- All responses must be consistent with each other
- The same accountReferenceId, productId, and user details must appear correctly in every response that references them
- IDs and references must match across responses — if login returns accountReferenceId "abc123", every other response that uses an account reference must use "abc123"
- Generate realistic-looking values, not placeholder text

Return only a JSON object where keys are the endpoint paths and values are the complete response bodies. No markdown, no backticks, no explanation.
```

Parse the result, save each endpoint response as a mock file, create the scenario JSON pointing to them.

---

## Focus Workflow

Focus is a Live Traffic feature that lets the developer mark specific intercepted requests as "interesting" for later use. Here is the exact flow:

### Step 1 — Developer clicks Focus on a request in Live Traffic
- The request is added to `state.focused` array in state.json
- The request object stored in focused includes the full request (method, path, headers, body, params) AND the full response body that was returned
- The focused request gets a unique ID
- In the UI, focused requests are visually marked in the Live Traffic stream

### Step 2 — Developer edits the response (optional)
- Clicking a focused request in Live Traffic shows the response in the JSON editor on the right
- The developer can edit the JSON directly or use the Tweak bar to ask Claude to modify it
- Changes are stored in memory against that focused request ID — nothing is saved to disk yet

### Step 3 — Developer saves
Saving is a deliberate action. There are three save options available from the Live Traffic toolbar when focused requests exist:

**Save as Mock** — saves the focused request as an individual mock file
- Converts the path to a folder name
- If no mock exists for that folder yet: save as `{mock-name}.json`
- If a mock already exists for that folder (collision): call Claude to generate a variant label from the request body, use that as the filename
- The saved mock file includes `_meta`, `_request`, `_headers`, and the response body
- The request body stored in `_request` is what the server uses later to match this variant in offline mode
- After saving, the mock appears in Mock Library under that endpoint

**Save as Template** — saves the API list from focused requests as a new scenario template
- Does NOT save response content — only the list of endpoints (method + path)
- Creates a new file in `templates/`
- Origin is set to `"captured"`

**Save as Scenario** — saves the focused requests as a complete captured scenario
- Saves each response as a mock file (same collision handling as Save as Mock)
- Creates a scenario JSON file pointing to those mock files
- Origin is set to `"captured"`
- The scenario appears in the Scenarios tab immediately

### Focus state is ephemeral
Focused requests live in memory and in `state.focused`. They are NOT automatically saved anywhere. The developer must explicitly choose Save as Mock, Save as Template, or Save as Scenario. Closing or restarting the server clears focused state.

---

## Serving — How Mocks Are Activated Per Endpoint

"Serving" means telling Charlotte which mock to return when a specific endpoint is called in offline mode. Only one mock can be serving per endpoint at a time.

### The serving map
`state.serving` is a flat map of endpoint path → mock file path:
```json
{
  "serving": {
    "/login": "login/login-standard.json",
    "/protected/375751/loyalty/accounts/digital-account-view/get-accounts": "protected-375751-loyalty-accounts-digital-account-view-get-accounts/miles-and-cash.json"
  }
}
```

### How serving is set
- In Mock Library, each mock row has a **"Serve this"** button. Clicking it sets that mock as serving for its endpoint — updates `state.serving` and writes `state.json`
- Only one mock per endpoint can be serving at a time — setting a new one replaces the old one
- The currently serving mock shows a filled radio button indicator in the UI

### How serving is cleared
- In Mock Library, the currently serving mock row shows a **"Stop serving"** button on hover. Clicking it removes that endpoint from `state.serving`
- After stopping, no mock is served for that endpoint in offline mode — Charlotte returns the no-mock error for that endpoint

### Serving and scenarios
- Activating a scenario sets the serving map in bulk — it replaces `state.serving` with the scenario's mock mappings for all its endpoints
- This is instant, no restart needed
- After activation, individual mocks can still be swapped out without affecting the scenario definition
- Deactivating a scenario clears the serving map entirely
- Only one scenario can be active at a time

### Serving is independent of scenarios
Once a scenario is activated and the serving map is set, the developer can freely change individual serving mocks in Mock Library without affecting the saved scenario. The scenario is a preset — activating it is like loading a save state. After loading, the developer can make changes freely.

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

## How Charlotte Integrates with the iOS App

Charlotte is NOT a proxy itself. It is a local HTTP server. The actual traffic interception is handled by Charles Proxy, which the developer already has running.

### iOS Setup (current, no code changes needed)
1. Developer runs Charlotte locally on port 3001
2. Charles Proxy is already running and intercepting iOS app traffic
3. Developer adds a **Map Remote** rule in Charles:
   - From: `https://api.yourcompany.com/*`
   - To: `http://localhost:3001/*`
4. Charles forwards matching requests to Charlotte
5. Charlotte logs the request, returns a mock (offline mode) or forwards to the real API (passthrough mode) and logs the response

In passthrough mode Charlotte needs to forward the request to the real API and return the response. Use axios for this — Charlotte sits between Charles and the real API.

Charlotte receives requests over **HTTP** (not HTTPS) from Charles Map Remote. Do not require HTTPS on the Charlotte server.

### Android Setup (future, not current priority)
Android cannot use Charles Map Remote the same way iOS can. For Android, the base URL in the Android codebase will need to be changed to point directly to the Charlotte server (e.g. `http://10.0.2.2:3001` for Android emulator, or the Mac's local IP for a physical device).

Important: Android requires **HTTP** (not HTTPS) when pointing directly at a local server. Charlotte must support plain HTTP. Do not redirect HTTP to HTTPS. When Android support is added, no SSL setup should be required on the Charlotte side.

Keep this in mind when building the server — do not hardcode any HTTPS assumptions. The server should work cleanly over HTTP from day one.

---

## Key Technical Notes

- Use Express for the HTTP server
- The proxy functionality can use `http-proxy` or `axios` to forward requests
- Store `state.json` in the project root, read on startup, write on every state change
- Mock files are JSON — always validate before saving, return `{ valid: false, error: "..." }` if invalid
- The `_meta`, `_request`, and `_headers` fields are Charlotte internals — strip them before sending to the iOS app
- For Claude calls, use `claude -p` as a subprocess via `execSync`. No API key or SDK needed — Claude Code must be installed on the developer's machine
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
