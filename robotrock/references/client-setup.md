# Client setup

## Shared client module

Always create one client per app and import it everywhere:

```typescript
// lib/robotrock.ts
import { createClient } from "robotrock";

export const robotrock = createClient({
  app: "budgeting-service",
  version: process.env.AGENT_VERSION,
  polling: {
    intervalMs: 2_000,
    timeoutMs: 30 * 60_000,
  },
});
```

## `createClient` options

| Option | Where | Description |
|--------|-------|-------------|
| `apiKey` | Client | API key string. Omit to read `ROBOTROCK_API_KEY` from env. |
| `app` | Client | Inbox bucket name in the RobotRock dashboard. |
| `version` | Client | Agent release version (semver, SHA, tag). Defaults to `AGENT_VERSION` / `ROBOTROCK_AGENT_VERSION` from env. |
| `advanced.contextVersion` | Client | Task context wire format (default `2`). Rarely changed. |
| `webhook` | Client | `{ url, headers? }` — applied to every action. Mutually exclusive with `polling`. |
| `polling` | Client | `{ intervalMs?, timeoutMs? }` — used when no webhook. Mutually exclusive with `webhook`. |

When `app` is omitted, the API uses your API key **name** as the inbox bucket.

## Environment variables

```bash
ROBOTROCK_API_KEY=rr_...           # required
ROBOTROCK_WEBHOOK_SECRET=rrwhsec_...  # required for webhook verification
ROBOTROCK_APP=my-service           # optional default app bucket
ROBOTROCK_BASE_URL=https://...     # optional, for local/self-hosted
AGENT_VERSION=1.2.0                # optional default agent version on client
```

Create API keys in the RobotRock dashboard under **Settings → API Keys**.

## Client methods

```typescript
// Create task (polls or returns immediately depending on client config)
await robotrock.sendToHuman({ type, name, actions, ... });

// Inspect or cancel without polling
await robotrock.getTask("task_...");
await robotrock.cancelTask("task_...");
```

## Mode selection

```typescript
// Polling mode — blocks until handled
export const robotrock = createClient({
  app: "my-service",
  version: "1.0.0",
  polling: { timeoutMs: 10 * 60_000 },
});

// Webhook mode — returns immediately with mode: "created"
export const robotrock = createClient({
  app: "my-service",
  version: "1.0.0",
  webhook: {
    url: "https://your-app.com/api/robotrock/webhook",
    headers: { "x-request-id": "optional" },
  },
});
```

You cannot set both `webhook` and `polling` on the same client.
