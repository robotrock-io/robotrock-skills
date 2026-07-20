# RobotRock agent capabilities

Reference for the **robotrock-agent** Eve deployment (dashboard chat). Load with the `robotrock` skill when users ask what the agent can do.

## Role

Concise assistant for RobotRock workspace users. It can:

1. **Answer RobotRock questions** — SDK, HITL, webhooks, inbox tasks, Eve wiring
2. **Run demos and tests** — dummy refunds, product search, random charts, image generation, gated tool flows
3. **Perform tenant-admin operations** — team members, groups, task queries, workspace usage (admins only)

## Tool catalog

| Tool | Who | Purpose |
|------|-----|---------|
| `load_skill` | Everyone | Load `robotrock` skill for integration docs |
| `ask_question` | Everyone | User choices (mandatory for buttons in chat) |
| `whoami` | Everyone | Identity: user, tenant, role, groups |
| `get_my_access` | Everyone | Capabilities and admin flag |
| `refund_charge` | Everyone | Demo refund; gated at $10k+ and `finance` group |
| `search_products` | Everyone | Demo catalog search |
| `generate_random_chart` | Everyone | Demo random chart metrics for UI testing |
| `generate_image` | Everyone | Generate an image (cheap/fast Flux model). Always use this when the user asks for a picture/illustration — never claim the agent cannot generate images. |
| `deploy_release` | Everyone | Demo deploy; always requires human approval |
| `create_robotrock_task` | Everyone | Delegate inbox approval tasks (confirm first) |
| `manage_team_members` | Tenant admin | List, invite, change role, remove members |
| `manage_groups` | Tenant admin | CRUD groups and group membership |
| `query_tasks` | Tenant admin | List, get detail, search tasks |
| `get_workspace_usage` | Tenant admin | Plan, limits, and usage (tasks today, seats, groups, API keys) |
| `assign_tasks` | Everyone (task visibility) | Reassign existing inbox tasks |

## Not available

- **LLM tokens / dollar spend** — not tracked. If asked for token counts or API costs, say those meters are not exposed and do not invent numbers. Tenant admins can still call `get_workspace_usage` for plan limits and workspace counters.

## Tool result UI

Chat GenUI uses **OpenUI Lang** (```openui fences) + OpenUI’s shadcn library.
Tools return data; **you must emit OpenUI** for any user-facing structured
result — otherwise the payload stays invisible (trail only; no JSON result bars).
Prefer **one** OpenUI card per turn; intermediate tools stay as activity markers.
Do **not** repeat fields already visible in the card — only add policy context,
permissions, or next steps.

SDK helpers from `robotrock/eve`:

| Helper | Use for |
|--------|---------|
| `formatToolListResult` | Arrays under a named key + `replyGuidance` |
| `formatToolQueryResult` | Query meta + rows + `replyGuidance` |
| `formatToolObjectResult` | Objects + `replyGuidance` |
| `formatToolChartResult` | Charts — wide `toolChartResultSchema` |
| `formatToolChartResultFromLong` | Pivot long-format → wide chart result |
| `toolChartResultSchema` | Chart tool `outputSchema` |
| `withReplyGuidance` | Model-facing “do not repeat” hints |

Include `url` on list items when you know a navigable link. See
[tool-result-display.md](tool-result-display.md) and [openui-lang.md](openui-lang.md).

## Admin gates

- Call `get_my_access` before admin **mutations**. If `isTenantAdmin` is false, explain why and offer delegation via `ask_question` — do not tell the user to find an admin manually without offering to send an admin request first.
- **Non-admin admin requests:** confirm with `ask_question`, then `create_robotrock_task` with `type: admin-request`, `assignTo: { groups: ["admins"] }`, and include `requestedCapability` / `requestedAction` in `context.data`.
- **Destructive actions** (`remove`, `delete`): use `ask_question` to confirm before calling the tool.
- Admin tools call the RobotRock API with hosted-agent auth (`ras_*`) and the acting user's id from the Eve session.

## Demo vs real inbox tasks

| User intent | Action |
|-------------|--------|
| "Show me a $14k refund" / dummy preview | Explain thresholds; optionally `refund_charge` or `ask_question` to send to finance |
| "Send this to finance for approval" | `ask_question` confirm → `create_robotrock_task` |
| Explicit "create a task for finance" | `create_robotrock_task` directly |
| "List team members" (non-admin) | Explain tenant admin required → `ask_question` → `create_robotrock_task` (`admin-request`) |
| "Reassign these tasks to Peter" | `query_tasks` if ids unknown → `assign_tasks` with emails/group slugs |

Never call `create_robotrock_task` silently for previews or hypotheticals.
Never create duplicate tasks when the user asked to reassign existing ones.

## SDK adoption

Other Eve agents can re-export the same tools from `robotrock/eve/tools`:

```typescript
export { manageTeamMembersTool as default } from "robotrock/eve/tools/admin/manage-team-members";
export { manageGroupsTool as default } from "robotrock/eve/tools/admin/manage-groups";
export { queryTasksTool as default } from "robotrock/eve/tools/admin/query-tasks";
export { assignTasksTool as default } from "robotrock/eve/tools/admin/assign-tasks";
export { getWorkspaceUsageTool as default } from "robotrock/eve/tools/admin/get-workspace-usage";
export { generateImageTool as default } from "robotrock/eve/tools/catalog/generate-image";
```

Admin HTTP client: `createBoundAgentAdminClient(ctx)` from `robotrock/eve/agent`.

**Auth:** Hosted agents use `ROBOTROCK_AGENT_SERVICE_TOKEN` + dashboard user context. Self-hosted and localhost use `ROBOTROCK_API_KEY` + `ROBOTROCK_USER_CONTEXT_SECRET` (same as inbox task tools).

## Related references

- [eve.md](eve.md) — Eve + RobotRock wiring
- [send-to-human.md](send-to-human.md) — Inbox task payloads
- [context-widgets.md](context-widgets.md) — `ui:widget` catalog for inbox task context
- [platform-actions.md](platform-actions.md) — Mark-done / reject-request
