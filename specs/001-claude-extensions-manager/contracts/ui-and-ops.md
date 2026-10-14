# Contract: `/ext` command, pane, and operations

## Slash command

| Command | Effect | Output |
|---|---|---|
| `/ext` | opens (or focuses) pane `ext-manager` | `{ text: "Extensions manager opened" }`; if the pane is not placed: `{ text: "Terminal too narrow: widen to ~110 columns and run /ext again" }` |

## Pane layout (terminal surface)

```text
Store: [~/src/Ivan________] [Change]          cwd: <project>   [Undo last] [Rescan]
Import: 3 imported, 1 skipped (name taken)    [Show report] [Undo import]
Filter: [______]  Type: [all|skill|plugin|mcp]  Scope: [global|project]  State: [all|...]
-- Skills ------------------------------------------------------------------
 [on ] lean-ctx        dev        global=link  project=-     Pick the right...
 [off] speckit-plan    dev        global=-     project=link  Execute the...
 [!! ] old-skill       seo        global=dangling            [remove link]
 [ro ] synced          managed    claude.ai synced skills
-- Plugins -----------------------------------------------------------------
 [on ] ponytail        imported   global=on    project=-     skills 5
-- MCP servers -------------------------------------------------------------
 [on ] lean-ctx        imported   global=registered          env: none
 [~~ ] n8n-ivan        imported   drift                      [sync from store]
Status: <last op: "applied now" | "restart to apply" | error>
```

- First open with no `storeRoot`: only the Store row, prefilled `~/src/Ivan`, and [Use this store].
- Each `[on ]`/`[off]` is a `Button` for the scope selected in Scope. Enter/Space toggles, Tab moves, Escape returns focus to the prompt.
- Secret values never enter the tree.

## Operations (argv via `$.process.run`, or `$.command.run`)

| Kind / scope | Activate | Deactivate | Live |
|---|---|---|---|
| skill / global | `ln -sfn <store> ~/.claude/skills/<n>` | `rm ~/.claude/skills/<n>` (link only) | watch |
| skill / project | `mkdir -p <cwd>/.claude/skills`; `ln -sfn <store> <cwd>/.claude/skills/<n>` | `rm` that link | watch |
| plugin / global | ensure global link; `claude plugin enable <n>@skills-dir --scope user --json` | `claude plugin disable ... --scope user --json` | `/reload-plugins` |
| plugin / project | ensure global link; if not active globally `disable --scope user`; `enable --scope local` (cwd) | `disable --scope local` | `/reload-plugins` |
| mcp / global | `claude mcp add-json -s user <n> <config>` | `claude mcp remove -s user <n>` | restart |
| mcp / project | `/mcp enable <n>` or `claude mcp add-json -s local <n> <config>` if not registered | `/mcp disable <n>` | native |
| mcp / project fallback | remove from `projects[cwd].disabledMcpServers` | add to it | restart |
| mcp / sync | `remove` + `add-json` at the registered scope from store config | n/a | restart |

Result per op: `{ ok, applied: "now" | "restart", message }`. Non-zero exit: `ok: false`, `message` = first stderr line, rescan, nothing else changed.

## Import steps (each with its inverse)

| Candidate | Steps | Inverse |
|---|---|---|
| real skill dir at `<loc>/<n>` | `mv <loc>/<n> <root>/skills/imported/<n>`; `ln -sfn` back | `rm` link; `mv` back |
| marketplace plugin `<n>@<m>` | `cp -a <installPath> <root>/plugins/imported/<n>`; global link; copy `pluginConfigs` and data dir; enable new id where old was enabled; `disable` old id there | `enable` old id; `disable` new id; `rm` link; `rm -r` store copy |
| MCP `<n>` without store file | write `<root>/mcp/imported/<n>/server.json` (0600) | `rm -r` that folder |

## Guards

- Refuse any op on `readonly` items and on this mod.
- Refuse a name taken in the store or in `~/.claude/skills/` by a different item.
- First write per file per session: backup to `~/.claude/backups/ext-manager/`.
- `~/.claude.json` fallback: re-read, compare `mtimeMs`, temp file + `mv`; on mismatch rescan and ask to retry.
