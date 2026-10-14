# Data Model: Claude Extensions Manager

Built in memory by one scan. Persisted: `storeRoot` (`$.store`), store items (disk), import logs (`<root>/.ext-manager/imports/`).

## Store

| Field | Type | Notes |
|---|---|---|
| `root` | string | default `~/src/Ivan` |
| `kinds` | `skills/`, `plugins/`, `mcp/` | each `<category>/<item>` |
| `health` | `ok` \| `missing` \| `unreadable` | missing: inventory read-only, offer to pick another |

## Item (one row in the pane)

| Field | Type | Notes |
|---|---|---|
| `key` | string | `${kind}:${name}` (store items) or `${kind}:${origin}:${name}` (outside) |
| `kind` | `skill` \| `plugin` \| `mcp` | |
| `name` | string | must match `^[A-Za-z0-9._-]+$` |
| `category` | string? | store items only |
| `storePath` | string? | absent while outside the store |
| `description` | string? | SKILL.md frontmatter, plugin manifest |
| `origin` | `store` \| `outside` \| `managed` | managed = Claude Code / policy / this mod |
| `activations` | Activation[] | where it is active |
| `state` | `active` \| `inactive` \| `broken` \| `drift` \| `readonly` | per shown scope |
| `reason` | string? | broken, readonly, drift, not imported |
| `secrets` | string[]? | MCP key names only |

## Activation

| Field | Type | Notes |
|---|---|---|
| `scope` | `global` \| `project` | project = session cwd |
| `method` | `link` \| `enabled-flag` \| `registration` | skill: link; plugin: global link + enabled flag; mcp: registration |
| `target` | string | link path or config location |
| `health` | `ok` \| `dangling` \| `drift` | dangling: link to a missing store item |

## State transitions

```text
outside  --import-->     store item, same activations (method may change)
inactive --activate-->   active     (applied: now | restart)
active   --deactivate--> inactive   (applied: now | restart)
broken   --deactivate--> inactive   (removes dangling link)
drift    --sync-->       active     (registration rewritten from store)
readonly : none
```

## ImportRun (`<root>/.ext-manager/imports/<ISO time>.json`)

| Field | Type |
|---|---|
| `at` | ISO time |
| `steps` | `{ item, action, argv \| fileOp, inverse }[]` in execution order |
| `skipped` | `{ item, reason }[]` |
| `undone` | boolean |

## Op (session undo log, memory only)

`{ item, action: activate | deactivate | sync, inverse, applied: now | restart | failed, at }`

## Relationships

Store 1..n Items. Item 0..n Activations. Plugin 1..n bundled skills / MCP servers (nested, read-only). ImportRun 1..n steps over Items.
