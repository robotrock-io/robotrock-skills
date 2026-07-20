# Eve agents (dashboard chat)

Integration for [Eve](https://eve.dev) deployments that chat in the RobotRock inbox.

**There is no `npx eve init robotrock` template.** Eve only scaffolds a generic agent:

```bash
npx eve init my-agent
cd my-agent
bun add robotrock eve
```

`eve init robotrock` would create an agent **named** `robotrock`, not a RobotRock-ready template. Wire RobotRock with the SDK steps below (new agent or existing deployment).

Load the **eve** skill (`node_modules/eve/docs/`) for Eve authoring; use this reference for RobotRock-specific wiring.

## SDK entry points (v1.1+)

| Import | Purpose |
|--------|---------|
| `robotrock/eve` | Input mapping, `buildTaskHandledResumeMessage` (no `eve` peer) |
| `robotrock/eve/agent` | Channel, auth, hooks, session client, inbox task APIs |
| `robotrock/eve/tools` | `createRobotrockTools()` bundle |
| `robotrock/eve/tools/inbox/create-task` | Inbox delegation (`create_robotrock_task`) |
| `robotrock/eve/tools/identity/whoami` | Dashboard user identity |
| `robotrock/eve/tools/identity/my-access` | Role and group capabilities |
| `robotrock/eve/tools/admin/manage-team-members` | Tenant-admin team CRUD |
| `robotrock/eve/tools/admin/manage-groups` | Tenant-admin group CRUD |
| `robotrock/eve/tools/admin/query-tasks` | Tenant-admin task list/search/detail |
| `robotrock/eve/tools/admin/get-workspace-usage` | Tenant-admin plan/limits/usage |
| `robotrock/agent-admin` | HTTP client for `/v1/agent-admin/*` (API key or `ras_*` + acting user) |

Tool results should use `formatToolListResult` / `formatToolObjectResult` / `formatToolQueryResult` from `robotrock/eve`. Dashboard GenUI uses the **[OpenUI](https://www.openui.com) spec** — the agent **must** emit OpenUI Lang (```openui fences) for user-visible structured data; tools return data only. Without OpenUI the payload stays invisible (activity trail only). Chart tools: `outputSchema: toolChartResultSchema` and `formatToolChartResult` (or `formatToolChartResultFromLong`) — wide series only. Include `url` on list items when you know a navigable link. See [tool-result-display.md](tool-result-display.md) and [openui-lang.md](openui-lang.md).

## Minimal file layout

After `eve init`, add thin re-exports (one line each):

```typescript
// agent/hooks/robotrock-ready.ts
export { robotrockReadyHook as default } from "robotrock/eve/agent";
```

```typescript
// agent/hooks/robotrock-chat-audit.ts
export { robotrockChatAuditHook as default } from "robotrock/eve/agent";
```

```typescript
// agent/channels/eve.ts
import { robotrockEveChannel } from "robotrock/eve/agent";
import { myWebappAuth } from "../lib/auth.js"; // optional — existing webapp

export default robotrockEveChannel({
  auth: [myWebappAuth()], // stack after RobotRock user-context auth
});
```

```typescript
// agent/tools/create_robotrock_task.ts
export { createInboxTaskTool as default } from "robotrock/eve/tools/inbox/create-task";
```

Optional tools:

```typescript
export { whoamiTool as default } from "robotrock/eve/tools/identity/whoami";
export { myAccessTool as default } from "robotrock/eve/tools/identity/my-access";
```

Customize inbox delegation:

```typescript
import { defineCreateInboxTaskTool } from "robotrock/eve/tools/inbox/create-task";

export default defineCreateInboxTaskTool({
  resolveDelegationReason(input, caller) {
    // optional business rules
    return "Routed to finance because …";
  },
});
```

## RobotRock-ready detection

On connect, RobotRock reads `GET /eve/v1/info`. The deployment is **RobotRock-ready** when the manifest includes the **`robotrock-ready`** hook slug (from `robotrockReadyHook` above). No manual toggle in Settings.

## Protected deployments (Bearer token)

If your Eve channel requires authentication, RobotRock must reach several routes with the **same Bearer token** the workspace admin enters under **Settings → Agents → Deployment access**. On the agent, set:

```bash
EVE_SELF_AUTH_TOKEN=<same-secret-as-dashboard>
```

`robotrockEveChannel()` registers `eveSelfServiceAuth()` so those requests are accepted. The token is stored in WorkOS Vault (never Convex) and sent as `Authorization: Bearer <token>` on every RobotRock → agent call below.

### Routes RobotRock calls (must accept the dashboard Bearer token)

| When | Method | Path | Notes |
|------|--------|------|-------|
| Connect / re-sync | `GET` | `/eve/v1/info` | Required — detects agent name + `robotrock-ready` |
| Connect / re-sync | `GET` | `/eve/v1/health` | Optional health probe |
| Connect / re-sync | `HEAD`/`GET` | `/favicon.ico`, `/icon.svg`, `/icon.png` | Optional avatar in Settings |
| Inbox chat (proxied) | `POST` | `/eve/v1/session` | Create chat session |
| Inbox chat (proxied) | `POST` | `/eve/v1/session/:sessionId` | Send message / task-handled resume |
| Inbox chat (proxied) | `GET` | `/eve/v1/session/:sessionId/stream` | SSE message stream |

During inbox chat, RobotRock also attaches `X-RobotRock-User-Token` (user-context JWT). That is separate from the deployment Bearer token — both may be required on the same request.

### Smoke-test with curl

Replace `https://my-agent.example.com` and `your-shared-secret`:

```bash
# Connect-time probe (must return agent manifest JSON)
curl -sS -H "Authorization: Bearer your-shared-secret" \
  https://my-agent.example.com/eve/v1/info

# Chat transport (must not return 401)
curl -sS -o /dev/null -w "%{http_code}\n" \
  -X POST -H "Authorization: Bearer your-shared-secret" \
  -H "Content-Type: application/json" \
  -d '{}' \
  https://my-agent.example.com/eve/v1/session
```

If `/eve/v1/info` returns 401 without the header but 200 with it, reconnect in Settings using **Bearer token** and paste the same value as `EVE_SELF_AUTH_TOKEN`.

SDK constants for these paths: `ROBOTROCK_EVE_CONNECT_ROUTES`, `ROBOTROCK_EVE_CHAT_ROUTES`, and `EVE_SELF_AUTH_TOKEN_ENV_VAR` from `robotrock/eve/agent`.

## Environment

### Self-hosted (one workspace per deployment)

Set after connecting in **Settings → Agents**:

| Variable | Purpose |
|----------|---------|
| `ROBOTROCK_API_KEY` | Workspace `ll_*` key |
| `ROBOTROCK_USER_CONTEXT_SECRET` | Verify dashboard user-context HMAC JWTs |
| `EVE_SELF_AUTH_TOKEN` | Same Bearer secret as **Settings → Agents → Deployment access** when `/eve/v1/*` requires auth |
| `ROBOTROCK_BASE_URL` | API base (optional; default production) |
| `ROBOTROCK_APP` | Default inbox app bucket |

### Multi-tenant (developer registry)

| Variable | Purpose |
|----------|---------|
| `ROBOTROCK_AGENT_SERVICE_TOKEN` | `ras_*` from **Developer → Agent deployments** |
| `ROBOTROCK_BASE_URL` | Optional; local dev (`http://localhost:4001/v1`) |

Platform user-context JWT verification fetches the public key from `/.well-known/robotrock-user-context-public-key` automatically.

## Existing Eve agent + custom webapp

RobotRock does **not** add a second channel. Extend the single `agent/channels/eve.ts` with `robotrockEveChannel({ auth: [existingAuth] })`. RobotRock uses `X-RobotRock-User-Token`; your webapp keeps its own `Authorization` / session auth. Sessions stay separate unless you share `sessionId` intentionally.

## Dashboard chat flow

1. Connect deployment URL in **Settings → Agents**.
2. Users chat via dashboard proxy (`/{tenant}/api/eve/{connectionId}`).
3. Gated tools and `ask_question` park with `input.requested` → inline buttons in chat.
4. `robotrockChatAuditHook` logs HITL choices to the audit trail (RobotRock sessions only).

For inbox tasks delegated to other people, use `create_robotrock_task` — see [send-to-human.md](send-to-human.md).

Tool results are domain data + optional `replyGuidance`. Rich UI is OpenUI Lang from the agent (```openui fences). See [tool-result-display.md](tool-result-display.md).

## `ask_question` (mandatory for choices)

Inline buttons appear only when Eve emits `input.requested` from `ask_question`. Prose option lists do **not** render buttons.

```ts
await ask_question({
  prompt: "Which path?",
  options: [
    { id: "polling", label: "Polling" },
    { id: "webhooks", label: "Webhooks" },
  ],
});
```

- ≤6 options → one button each
- After calling `ask_question`, do not repeat options in assistant text
- If the user sends other messages while a question panel is still open, RobotRock
  **closes** older question panels — only the latest open question can be answered.
  If you still need that input, call `ask_question` again; do not assume the user
  will answer a closed panel.

## Parallel tool approvals (confirmation batch)

Eve parks each gated tool call separately, but emits one `input.requested` with a
`requests[]` batch. Channels (Chat SDK) render that as **one card**. RobotRock
agent chat matches that:

- Consecutive confirmation approvals in the same assistant turn coalesce into
  **one** action panel (e.g. 3× `deploy_release` → `Approve: Deploy a release (3)`).
- When they share the same Approve/Deny options, one Submit answers every
  `requestId` via a single `send({ inputResponses })`.
- Siblings in that batch stay open together (they are not closed against each other).
- Older *question* panels across turns still close; superseded-by-later-message
  rules are unchanged.

## Eve input display modes

| Eve `display` | Meaning | Chat UI |
|---------------|---------|---------|
| `confirmation` | Gated tool approval | Approve / Deny (batched when parallel) |
| `select` | `ask_question` with options | Buttons or radio form |
| `text` | Open `ask_question` | Text field |

## Checklist

- [ ] `bun add robotrock` (and `eve` if not already)
- [ ] `robotrock-ready` hook re-export
- [ ] `robotrockEveChannel` in `agent/channels/eve.ts`
- [ ] Env vars for deployment kind (self-hosted vs multi-tenant)
- [ ] Protected deployment: `EVE_SELF_AUTH_TOKEN` matches dashboard Bearer token; curl `/eve/v1/info` with Bearer returns 200
- [ ] Optional: `robotrock-chat-audit` hook, inbox/identity tools
- [ ] Use `ask_question` for user choices; gate dangerous tools in Eve
- [ ] Connect deployment in RobotRock **Settings → Agents**

Reference implementation: [robotrock-agent](https://robotrock-agent.vercel.app) ([docs](https://docs.robotrock.io/agents/build)).
