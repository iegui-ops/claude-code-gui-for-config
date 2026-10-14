# Research: Claude Extensions Manager

Findings from this machine (Claude Code 2.1.300, mod API of build 2.1.299) on 2026-10-15. Items marked **verified** were tested here; the probes were undone afterwards.

## R1. Delivery as a mod

- **Decision**: The repository root is the mod `ext-manager`: `.claude-plugin/plugin.json`, `hooks/hooks.json`, `hooks/register.tsx`, `types/index.d.ts`. Slash command `/ext` opens a pane (`$.ui.open` + `ui.render` on `Pane`) drawn with the terminal surface's `Box`, `Text`, `Button`, `Input`, `Select`.
- **Rationale**: User's choice (clarification). The API covers what is needed: `$.fs` (read, write, list with `isLink`, stat), `$.process.run` (argv, no shell), `$.settings.read`, `$.command.list/run/register`, `$.store` (persists across sessions).
- **Name**: `/ext`; `/plugin`, `/mcp`, `/skills` are built-ins and refused.

## R2. Store layout and root

- **Decision**: `<root>/skills/<category>/<item>`, `<root>/plugins/<category>/<item>`, `<root>/mcp/<category>/<item>`. Root kept in `$.store` (`storeRoot`), default `~/src/Ivan`. The existing library `~/src/Ivan/skills/<category>/<skill>` already fits; `plugins/` and `mcp/` are created on first import. Anything else under the root (other projects in `~/src/Ivan`) is ignored.
- **Rationale**: Clarification answer (one folder per kind); default root avoids migrating any skill.
- Manager metadata lives in `<root>/.ext-manager/` (import logs).

## R3. Skills

- **Decision**: Activate = `ln -sfn <store item> <scope>/.claude/skills/<name>` (global `~/.claude/skills/`, project `<cwd>/.claude/skills/`); deactivate = `rm` of that link, only when the entry `isLink`. Via `$.process.run`.
- **Live apply**: Claude Code watches skill dirs; reported "applied now", verified in quickstart.
- **Read-only, never imported**: `~/.claude/skills/synced/` (claude.ai synced skills, managed by Claude Code), dot entries (`.trash`), skills bundled inside plugins.

## R4. Plugins (verified)

Facts tested here:
1. A plugin folder linked into `~/.claude/skills/<name>` loads as `<name>@skills-dir`, **enabled by default**.
2. The same link in a project's `.claude/skills/` is **not** loaded (also in a trusted project).
3. With the global link, `claude plugin disable <id> --scope user` + `claude plugin enable <id> --scope local` (run in the project) gives: enabled in that project, disabled elsewhere.

- **Decision**: Every store plugin that is active anywhere has exactly one link, global: `~/.claude/skills/<name>` → `<root>/plugins/<cat>/<name>`. Scope is decided by `enabledPlugins`, written only through `claude plugin enable|disable <id> --scope user|local --json`.
  - Activate global: ensure link, `enable --scope user`.
  - Activate project only: ensure link, `disable --scope user` (if not active globally), `enable --scope local`.
  - Deactivate: `disable` at that scope. The link stays, because other projects may enable it in their own local settings that this manager cannot see; a disabled plugin costs nothing.
  - Live: `$.command.run({ command: "reload-plugins" })` when listed by `$.command.list()`.
- Plugins and skills share `~/.claude/skills/`: a name used by both is a collision, refused and reported.
- **Alternatives**: link per project (rejected: not loaded, fact 2); local directory marketplace (rejected: needs a marketplace manifest kept in sync, no gain over fact 3).

## R5. Importing marketplace plugins

- **Decision**: copy `installPath` to `<root>/plugins/imported/<name>`, link it globally, reproduce the old enabled state for the new id `<name>@skills-dir` (user scope, and local scope for the current project), copy `pluginConfigs["<old id>"]` to the new id, then **disable** (not uninstall) the old `<name>@<marketplace>` at every scope where it was enabled.
- **Rationale**: Disabling keeps the original install, so undo is exact: re-enable old id, drop the new id's entries, remove link and store copy. Uninstall would force a re-download of a possibly different version.
- **Consequences** (accepted in spec): no more marketplace updates for the store copy; the plugin id changes, so a plugin's saved data dir under `~/.claude/plugins/data/` is keyed by the old id. Copy it to the new id's dir during import; exact naming of that dir is checked in the first implementation task.

## R6. MCP servers

| Origin | Location | Project-level off switch |
|---|---|---|
| user | `~/.claude.json` `mcpServers` | `projects[<cwd>].disabledMcpServers` |
| local | `~/.claude.json` `projects[<cwd>].mcpServers` | same |
| project | `<cwd>/.mcp.json` | `disabledMcpjsonServers` |
| plugin | plugin's `.mcp.json` | name `plugin:<plugin>:<server>` in `disabledMcpServers` |

- **Link not used for MCP (deliberate)**: an MCP can only be "linked" by wrapping it in a plugin folder, and then Claude Code renames it to `plugin:<p>:<s>` and its tools from `mcp__<s>__*` to `mcp__plugin_<p>_<s>__*` (seen on disk: `plugin:telegram:telegram`). The user's hooks and permission rules match `mcp__lean-ctx__*`, so the rename breaks them. Spec FR-015 allows registration when a link is not usable; this is that case.
- **Decision**: store item = `<root>/mcp/<cat>/<name>/server.json` `{ "name", "config" }` (dir `0700`, file `0600`, holds secrets). Registration = the Claude Code config entry, derived from the store file:
  - Activate global: `claude mcp add-json -s user <name> '<config>'`. Deactivate global: `claude mcp remove -s user <name>`.
  - Activate for project: if registered at user scope, remove `<name>` from `disabledMcpServers`; otherwise `claude mcp add-json -s local <name> '<config>'` in cwd. Deactivate for project: add to `disabledMcpServers`.
  - Project toggles prefer `$.command.run({ command: "mcp", args: "enable|disable <name>" })` (native, live). If that form is refused, edit `~/.claude.json` with the guard in R8, and report "restart to apply".
- **Drift**: if a registered config differs from its store file, the row shows "drift" with a "sync from store" action; the store wins.
- **Import**: for each server not in the store, write its store file from the current config. The registration stays as it is, so nothing stops working. `.mcp.json` entries stay in the project repo.
- **Out of scope / ignored**: claude.ai connectors (spec), and the undocumented `mcpServers` key in `~/.claude/settings.json` (`n8n-*`).

## R7. Automatic import

- **Decision**: runs after the store root is chosen and on every scan. It collects the candidates (real skill dirs in global/project skill locations, marketplace plugins, MCP servers without a store file), backs up the files it will touch, executes, and writes `<root>/.ext-manager/imports/<ISO time>.json` with every step and its inverse. The pane shows the report and an "Undo import" button that replays the inverses in reverse order.
- **Skipped and shown read-only**: R3 read-only list, policy-managed servers, claude.ai connectors, this mod (`$.plugin.name`), names already taken in the store or in `~/.claude/skills/` ("not imported: name taken").
- Default category for imports: `imported`.

## R8. Safety

- Writes go through official writers (`claude plugin`, `claude mcp`, `/mcp`, `/reload-plugins`) plus `ln`, `rm` of links, `mv`/`cp` into the store. The only hand edit is the `~/.claude.json` fallback: re-read, compare `mtimeMs` with the scan, write a temp file, `mv` over it.
- Before the first write of a session, each touched config file is copied to `~/.claude/backups/ext-manager/<name>.<ISO time>`.
- Every item name is checked against `^[A-Za-z0-9._-]+$` before use in a path.
- Secret values (MCP `env`, `headers`) never enter the UI tree.

## R9. Testing

- `claude plugin validate .`, `tsc -p .`, `claude plugin test .` with two fixture-based tests: `inventory.test.ts` (raw listings to items) and `import.test.ts` (inventory to import steps plus inverses), then the manual [quickstart](./quickstart.md).

## Open points

- claude.ai connectors stay out of scope; on disk the user disables them per project often, and they use the same `disabledMcpServers` list. Easy to add later.
