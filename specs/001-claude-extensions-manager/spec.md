# Feature Specification: Claude Extensions Manager

**Feature Branch**: `001-claude-extensions-manager`

**Created**: 2026-10-15

**Status**: Draft

**Input**: User description: "a gui to see, activate and deactivate plugins, skills and MCPs in a claude installation"

## Clarifications

### Session 2026-10-15

- Q: Which Claude installations should the manager cover? → A: Claude Code only (user and project configuration).
- Q: How do you open and use the manager? → A: As a Claude Code mod: an interactive pane inside Claude Code, opened with a slash command, not a separate app or web page.
- Q: When should a toggle take effect? → A: Live in the current session where possible, otherwise "restart to apply".
- Q: Which projects can the manager work on at project scope? → A: Only the current session's project, plus global.
- Q: What happens to items outside the store? → A: Imported into the store automatically, old location replaced by a link or registration; each run reported and undoable.
- Q: Store layout? → A: One folder per kind (`skills/`, `plugins/`, `mcp/`) with category folders inside.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See everything installed at a glance (Priority: P1)

From inside a Claude Code session, the user runs the manager's slash command and a pane opens showing one inventory of every plugin, skill and MCP server known to their Claude installation, grouped by type, each showing its name, short description, scope (global or a specific project), where it comes from (store, plugin, Claude Code managed) and whether it is currently active.

**Why this priority**: Today this information is spread across several settings files and folders; just seeing it in one place already answers "why is this skill loading?" and "what MCPs are eating my context?". Read-only is also zero risk.

**Independent Test**: With a known installation (e.g. 3 global skills, 1 project skill, 2 plugins with one disabled, 2 MCP servers), open the manager and confirm every item appears with the correct type, scope and state.

**Acceptance Scenarios**:

1. **Given** an installation with global skills, project skills, enabled and disabled plugins and MCP servers, **When** the user opens the manager, **Then** every item appears exactly once in its group with the correct active/inactive state.
2. **Given** the list is shown, **When** the user types in a search box or filters by type, scope or state, **Then** only matching items remain visible.
3. **Given** a skill is broken (its link points to a missing folder or its definition file is unreadable), **When** the list loads, **Then** the item is shown flagged as broken with the reason, instead of being hidden or crashing the view.

---

### User Story 2 - Activate and deactivate items (Priority: P2)

The user flips a toggle on any plugin, skill or MCP server to turn it on or off for a given scope, and the change takes effect without hand-editing any file: immediately in the current session where Claude Code allows it, otherwise from the next session.

**Why this priority**: This is the core action requested, but it depends on the inventory (P1) being correct first.

**Independent Test**: Deactivate one global skill, one plugin and one MCP server via the manager, start a new Claude session and confirm none of the three is available; reactivate them and confirm they come back.

**Acceptance Scenarios**:

1. **Given** an active global skill, **When** the user deactivates it, **Then** it stops being offered to new Claude sessions, its source files remain untouched in the store, and the manager shows it as inactive.
2. **Given** an inactive skill that exists in the store, **When** the user activates it globally or for the current project, **Then** it becomes available in new sessions at exactly that scope.
3. **Given** an enabled plugin, **When** the user disables it, **Then** the plugin is marked disabled in the installation's settings and the plugin itself stays installed.
4. **Given** an MCP server defined for the user or a project, **When** the user disables it, **Then** it no longer starts in new sessions for that scope, and its configuration is preserved so re-enabling restores it unchanged.
5. **Given** a toggle change, **When** it is applied, **Then** the manager tries to apply it to the current session and shows per item either "applied now" or "restart to apply"; other running sessions always need a restart.

---

### User Story 3 - Work per project (Priority: P3)

The user sees the effective set of extensions for the project of the current session (its working folder): what is inherited from the global scope, what the project adds, and what the project turns off, and can toggle items at project scope only.

**Why this priority**: The user keeps niche skills (n8n, speckit, supabase) per project rather than global; managing that by hand is the most error-prone part, but the global view already covers the common case.

**Independent Test**: Open the manager in a session started in a project that has project-only skills and a project MCP file; confirm the view distinguishes inherited vs project items, and that toggling at project scope does not change the global state.

**Acceptance Scenarios**:

1. **Given** a session started in a project, **When** the view loads, **Then** each item shows whether it comes from global or project scope.
2. **Given** a session started in a project, **When** the user activates a store skill for that project only, **Then** it is available in sessions started in that project and not in others.

---

### User Story 4 - One central store, activation by link (Priority: P2)

All skills, plugins and MCP servers the user owns live as subdirectories of one store directory that the user selects. Activating an item at a scope (global or the current project) creates a link from that scope's location to the item in the store whenever Claude Code can load the item through a link; only when it cannot does the manager register the item another way. Deactivating removes that link or registration and never touches the store.

**Why this priority**: It makes the store the single source of truth: one copy of each item, versioned and backed up where the user wants, shared by every project through links instead of duplicated folders.

**Independent Test**: Select a store directory containing one skill, one plugin and one MCP server; activate each for the current project; confirm each project location holds a link into the store (or, for items that cannot be linked, a registration pointing at the store), and that deactivating leaves the store byte-for-byte unchanged.

**Acceptance Scenarios**:

1. **Given** no store selected yet, **When** the user opens the manager, **Then** it asks for the store root (suggesting `~/src/Ivan`, whose `skills/` folder is the current library) and remembers the choice across sessions.
2. **Given** a selected store, **When** the user changes it, **Then** the inventory is rebuilt from the new store and existing links into the old store are listed so the user can see they still point there.
3. **Given** a skill in the store, **When** the user activates it for the current project, **Then** the project's skills location contains a link to the store item and no copy.
4. **Given** a plugin or MCP server in the store, **When** the user activates it, **Then** it is linked if Claude Code can load it from a link at that scope, otherwise registered with Claude Code pointing at the store item, and the row says which of the two was used.
5. **Given** an item active somewhere, **When** the user looks at its row, **Then** it shows every scope where it is active and where each link points.
6. **Given** items that exist outside the store (e.g. a plugin installed from a marketplace, an MCP server defined only in configuration, a skill folder copied into a project), **When** a store is selected for the first time or the manager finds new outside items on a later scan, **Then** it imports them into the store automatically: each item is moved into its kind's folder, its old location is replaced by a link or registration pointing at the store so it keeps working with the same active state, and a report lists what was imported.
7. **Given** an automatic import just ran, **When** the user chooses to undo it, **Then** every imported item returns to its original location and state.

---

### Edge Cases

- A settings file contains invalid content: the manager reports which file and why, shows what it could read, and refuses to write to that file until it is fixed.
- A settings file is changed outside the manager while it is open (e.g. by Claude itself or an editor): the manager detects it before writing and reloads rather than overwriting the newer content.
- The same skill name exists at global and project scope: both are listed with their scope; which one wins is shown.
- A real skill or plugin folder (not a link) inside an active-skills location: imported into the store and replaced by a link (FR-018); if the import fails it stays in place, listed with the reason, and is never deleted.
- An MCP server is provided by a plugin rather than configured directly: it is shown under that plugin and toggled through the plugin.
- The terminal is too narrow to place the pane: the manager says so (e.g. via a short message) instead of failing silently.
- The manager toggles itself off: it is listed as read-only so the user cannot disable the tool they are using.
- MCP servers managed by an organization or enterprise policy: shown as read-only.
- The selected store directory does not exist, is not readable, or is on a path that disappears (unmounted disk): the manager says so, keeps the previous inventory read-only, and offers to pick another store.
- A link at a scope points into the store but the item was renamed or deleted there: shown as broken with the old target; deactivating removes the dangling link.
- An item in the store has a name that collides with a different item already at the target location (a real folder or a link elsewhere): activation is refused with the reason; nothing is overwritten.
- Folders that Claude Code manages itself inside the global skills location (e.g. synced account skills): never touched, shown read-only.
- A project is moved or opened from another machine: links are absolute to the store, so they break if the store path differs; shown as broken.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-000**: System MUST be delivered as a Claude Code mod: a slash command opens an interactive pane in the running session, usable with the keyboard alone, and the pane can be closed without affecting the session.
- **FR-001**: System MUST discover and list all skills, plugins and MCP servers from the user's Claude installation, at global scope and at the scope of the current session's project.
- **FR-002**: System MUST show for each item: type, name, description (when available), scope, origin, and state (active, inactive, broken, read-only).
- **FR-003**: Users MUST be able to search by name/description and filter by type, scope and state.
- **FR-004**: Users MUST be able to activate and deactivate skills, plugins and MCP servers at global or project scope with a single action per item.
- **FR-005**: Deactivating a skill MUST NOT delete its source files; deactivating a plugin MUST NOT uninstall it; deactivating an MCP server MUST NOT lose its configuration (for store items, the store copy is the configuration).
- **FR-006**: System MUST list every item in the store that is not currently active at the shown scopes, so it can be activated.
- **FR-013**: Users MUST be able to select the store root from the manager; the choice MUST persist across sessions and default to `~/src/Ivan` (so the existing library `~/src/Ivan/skills/<category>/<skill>` is already the store's `skills/` folder and needs no migration).
- **FR-014**: The store MUST be organized with one folder per kind (`skills/`, `plugins/`, `mcp/`) under the store root, each holding category folders that hold the items (`<root>/<kind>/<category>/<item>`); the manager reads only these three folders and ignores anything else under the root. Imported items with no known category go to an `imported` category.
- **FR-018**: System MUST automatically import into the store every skill, plugin and MCP server found outside it (first store selection and every later scan), keeping each item's active state by replacing its old location with a link or registration to the store item.
- **FR-019**: Each import run MUST be backed up first, reported (item, from, to), and undoable as one unit.
- **FR-020**: Import MUST skip and show read-only: items Claude Code manages itself (synced account skills, organization-managed servers), claude.ai connectors, and this manager itself.
- **FR-015**: Activation MUST create a link from the scope's location to the store item whenever Claude Code can load that kind of item through a link at that scope; only otherwise MAY it register the item with Claude Code, and that registration MUST reference the store item, never a copy.
- **FR-016**: Deactivation MUST remove only the link or registration it created; the store MUST remain unchanged by any activate or deactivate.
- **FR-017**: System MUST never create a copy of a store item in a project or global location.
- **FR-007**: System MUST only change the specific entries it toggles and preserve all other content and formatting-relevant data in the files it edits.
- **FR-008**: System MUST keep a backup of each configuration file before its first modification in a session, so a change can be undone.
- **FR-009**: System MUST detect external changes to a file since it was read and refuse to overwrite them silently.
- **FR-010**: System MUST attempt to apply each change to the current session (reload skills, plugins or MCP connections) when Claude Code supports it, and MUST tell the user per item whether it was applied now or needs a restart; other running sessions are never changed live.
- **FR-011**: System MUST run locally on the user's machine and MUST NOT send installation data or configuration (which may contain tokens in MCP settings) to any external service.
- **FR-012**: System MUST NOT display secret values found in MCP configuration (API keys, tokens) in clear text.

### Key Entities

- **Store**: the user-selected directory holding one subdirectory per owned item; single source of truth. Attributes: path, layout, last scan.
- **Skill**: a named capability with a description; lives in the store and is active at a scope when it is linked into that scope's skills location. Attributes: name, description, store path, scopes where active, health.
- **Plugin**: a package that may bundle skills, commands, agents, hooks and MCP servers; either a store item (activated by link or registration) or installed from a marketplace. Attributes: name, origin (store or marketplace), version, active scopes, bundled contents.
- **MCP Server**: an external tool provider; either a store item (its definition kept in its own store subdirectory) or defined only in configuration. Attributes: name, scope, origin, connection type, active scopes, secret fields (masked).
- **Activation**: the link or registration that makes a store item active at one scope. Attributes: item, scope, method (link or registration), target path, health.
- **Scope**: global (user) or a specific project folder; determines where an item is active.
- **Configuration Source**: a file or folder the manager reads or writes; has a last-read fingerprint and a backup.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The user can find out whether any given skill, plugin or MCP server is active, and at which scope, in under 10 seconds from opening the manager.
- **SC-002**: Toggling an item takes one action and under 5 seconds, versus several minutes of locating and hand-editing files today.
- **SC-003**: 100% of items reported by Claude itself (its own skills/plugins/MCP listing) for a given scope appear in the manager with the same state.
- **SC-004**: Zero configuration entries are lost or altered other than the toggled one across 50 consecutive toggle operations (verified by comparing files before and after).
- **SC-005**: Any change made through the manager can be reverted to the exact previous state.
- **SC-006**: After any number of activations across projects, each store item exists exactly once on disk (zero copies outside the store).
- **SC-007**: For every kind Claude Code can load through a link, 100% of activations made by the manager are links.

## Assumptions

- Single user on their own Linux machine; no authentication or multi-user support.
- "Claude installation" means Claude Code configuration only: the user's home configuration plus per-project configuration. The Claude desktop app's MCP servers and claude.ai account connectors are out of scope and are not listed.
- The store generalizes the user's existing convention (skills in `~/src/Ivan/skills/<category>/<skill>`, activated by links; active-skills locations only contain links) to plugins and MCP servers, in `~/src/Ivan/plugins/` and `~/src/Ivan/mcp/`.
- Accepted trade-off of automatic import: a marketplace plugin moved into the store stops receiving marketplace updates; updating it becomes a manual action outside this feature.
- MCP servers have no folder of their own today; importing one creates its store folder holding its definition (secrets included, so the store must stay private: it is never published by the manager).
- Claude Code can load a plugin folder placed (or linked) in a skills location, so plugins, and MCP servers packaged as plugins, are expected to be linkable; this is verified per kind and scope during planning.
- Installing or updating plugins from marketplaces, and authoring new skills, plugins or MCP definitions, are out of scope; the manager activates, deactivates and organizes what exists.
- Changes are applied live to the session hosting the manager only where Claude Code exposes a way to do so; other running sessions pick them up on restart. A failed live apply never rolls back the saved change.
- Project scope means the project of the session hosting the manager; other projects are managed by opening the manager in a session there. No project picker and no disk scan.
