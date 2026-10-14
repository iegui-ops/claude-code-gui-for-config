# Implementation Plan: Claude Extensions Manager

**Branch**: `001-claude-extensions-manager` (no git repo yet) | **Date**: 2026-10-15 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-claude-extensions-manager/spec.md`

## Summary

A Claude Code mod: `/ext` opens a pane listing skills, plugins and MCP servers for global scope and the session's project. Everything the user owns lives in a selectable store (`<root>/{skills,plugins,mcp}/<category>/<item>`, default root `~/src/Ivan`); anything found outside is imported automatically, reported and undoable. Activation is a symlink wherever Claude Code loads one (skills at both scopes; plugins via one global link plus enabled flags per scope) and a registration derived from the store for MCP servers (linking would rename their tools). All config writes go through Claude Code's own CLI and commands. See [research.md](./research.md).

## Technical Context

**Language/Version**: TypeScript (TSX), compiled by Claude Code's mod engine (build 2.1.299+)

**Primary Dependencies**: none; only the `claude-code` mod API (`$.ui`, `$.fs`, `$.process`, `$.command`, `$.store`, `$.settings`)

**Storage**: the store on disk; `$.store` for `storeRoot`; import logs in `<root>/.ext-manager/imports/`

**Testing**: `claude plugin validate`, `tsc -p .`, `claude plugin test` (`inventory.test.ts`, `import.test.ts`), manual [quickstart.md](./quickstart.md)

**Target Platform**: Claude Code terminal surface on Linux

**Project Type**: Claude Code mod (plugin with hooks module)

**Performance Goals**: scan under 1 s for ~200 items; toggle feedback under 5 s (SC-002)

**Constraints**: argv only (no shell); secrets never in the UI; links-only deletion outside the store; never write a file changed since read; MCP names and tool names must not change

**Scale/Scope**: one user, tens of items per kind

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

No `.specify/memory/constitution.md`. Checked against the user's global rules (CLAUDE.md, solution-efficiency ladder):

- [x] **Simplicity**: no dependencies; official writers instead of hand-editing config; MCP stash replaced by the store file itself.
- [x] **Security**: argv-only processes, name validation before path use, secrets masked, MCP store files `0600`, local only.
- [x] **Data safety**: backups before first write, mtime guard, per-op and per-import undo; marketplace plugins disabled, not uninstalled, so undo is exact.
- [x] **Skill convention**: `~/.claude/skills/` ends up holding only links (plus `synced/`, managed by Claude Code); the store is the source of truth.

**Result**: PASS (pre- and post-design).

## Project Structure

### Documentation (this feature)

```text
specs/001-claude-extensions-manager/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/ui-and-ops.md
└── tasks.md             # /speckit-tasks
```

### Source Code (repository root)

```text
.claude-plugin/plugin.json   # name "ext-manager", types
hooks/
├── hooks.json               # { "modules": ["./register.tsx"] }
├── register.tsx             # /ext command, pane render, button handlers
├── inventory.ts             # pure: store listing + config JSON -> Item[]
├── importer.ts              # pure: Item[] -> import steps with inverses
├── ops.ts                   # run ops/steps (process, fs), backups, undo log
├── inventory.test.ts
└── importer.test.ts
types/index.d.ts             # PluginState contract for $.state atoms
```

**Structure Decision**: the repo root is the mod, loaded with `claude --plugin-dir .` while developing. `inventory.ts` and `importer.ts` stay pure (data in, data out) so the risky logic is tested without a session; `ops.ts` is the only file that touches disk or runs commands.

## Complexity Tracking

| Deviation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| MCP activated by registration, not link (spec prefers links) | a link needs a plugin wrapper, which renames the server and its tools | breaks the user's hooks and permissions on `mcp__lean-ctx__*` |
| plugin project scope via global link + local enable | Claude Code ignores plugin links in project skill dirs (verified) | per-project link does not load |
