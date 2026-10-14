# Quickstart: validate the Extensions Manager

## Prerequisites

- Claude Code 2.1.299+ (`claude --version`)
- Repo at `/home/xir/src/Ivan/gui-claude` (the repo root is the mod)
- For a safe first run, use a throwaway store root (e.g. `~/tmp/ext-store` with a copy of one skill) before pointing it at `~/src/Ivan`.

## Static checks

```bash
claude plugin validate /home/xir/src/Ivan/gui-claude
tsc -p /home/xir/src/Ivan/gui-claude
claude plugin test /home/xir/src/Ivan/gui-claude
```

Expected: no errors; `inventory.test.ts` and `import.test.ts` pass.

## Run

```bash
cd /home/xir/src/Ivan/gui-claude && claude --plugin-dir /home/xir/src/Ivan/gui-claude
```

Type `/ext`.

## Scenarios

1. **Store selection**: first open asks for the root, prefilled `~/src/Ivan`. Reopen in a new session: not asked again.
2. **Import report**: after choosing the store, the report lists marketplace plugins (ponytail, telegram, frontend-design) and MCP `lean-ctx` as imported; `synced` and this mod as skipped. Check `ls <root>/plugins/imported <root>/mcp/imported`, `ls -la ~/.claude/skills` (links only, plus `synced`).
3. **Nothing broke**: `claude plugin list --json` shows the `@skills-dir` ids with the same enabled state the old ids had; `claude mcp list` unchanged; tool names still `mcp__lean-ctx__*`.
4. **Undo import**: press Undo import; old ids enabled again, links and store copies gone, `claude plugin list --json` equals the pre-import snapshot.
5. **Skill toggle**: deactivate a global skill: link gone, store item intact; reactivate.
6. **Plugin project scope**: activate a store plugin for this project only: active here, `enabled: false` from `/tmp`.
7. **MCP global toggle**: deactivate `lean-ctx` globally; `claude mcp list` lacks it; activate; `jq '.mcpServers["lean-ctx"]' ~/.claude.json` equals `config` in its `server.json`.
8. **Drift**: edit a registered MCP by hand; row shows drift; sync restores the store version.
9. **Secrets**: no MCP row shows env or header values; `stat -c %a <root>/mcp/*/*/server.json` is `600`.
10. **Integrity (SC-004, SC-006)**: snapshot configs, do 10 toggle/undo pairs: only toggled entries changed; `find ~/.claude/skills <cwd>/.claude/skills -mindepth 1 -maxdepth 1 ! -type l` shows only `synced` and dot entries.
11. **Narrow terminal**: below ~110 columns `/ext` answers with the widen message.
