# Task context

`context` provides metadata shown to reviewers in the task detail view. Assignment (`assignTo`) is a **separate** top-level field — see [send-to-human.md](send-to-human.md).

## Shape

```typescript
context: {
  data: Record<string, unknown>; // required when context is present
  ui?: Record<string, unknown>;  // optional display customization
}
```

## Example

```typescript
await robotrock.sendToHuman({
  type: "budget-approval",
  name: "Approve Q4 budget update",
  context: {
    data: {
      requestId: "bud-2026-q4-17",
      department: "finance",
      requestedIncreaseUsd: 120000,
      submittedBy: {
        name: "Avery Chen",
        email: "avery@example.com",
      },
    },
    ui: {
      requestId: { "ui:title": "Request ID", "ui:widget": "string" },
      requestedIncreaseUsd: {
        "ui:title": "Requested increase (USD)",
        "ui:widget": "currency",
        "ui:options": { currency: "USD" },
      },
      submittedBy: {
        name: { "ui:title": "Submitted by" },
        email: { "ui:widget": "string" },
      },
    },
  },
  actions: [
    { id: "approve", title: "Approve" },
    { id: "reject", title: "Reject" },
  ],
});
```

## JSON Forms conventions

- `context.data` — raw values for reviewers
- `context.ui` — controls labels, widgets, placeholders, options

Common UI keys: `ui:widget`, `ui:title`, `ui:description`, `ui:placeholder`, `ui:options`

## Widget catalog

For the full list of available `ui:widget` values, when to override auto-selection, data shapes, and `ui:options`, see [context-widgets.md](context-widgets.md).

## Widget auto-selection

If `ui:widget` is omitted, RobotRock selects by value type:

| Type | Default widget |
|------|----------------|
| string | `string` |
| number | `number` |
| boolean | `boolean` |
| object | `object` |
| array of primitives | `list` |
| array of objects | `table` |

## Action schemas

Action `schema` and `ui` follow the same JSON Forms pattern for reviewer form fields on each action button. See [send-to-human.md](send-to-human.md).
