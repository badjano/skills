---
name: unity-mcp-project-settings
description: Saves and updates Unity MCP configuration only inside Unity projects. Use when configuring Unity MCP, the Unity relay, or UserSettings/mcp.json; never writes Unity MCP settings globally or into non-Unity projects.
---

# Unity MCP Project Settings

Configure Unity MCP at the project level only.

## Scope guard

Before writing anything, verify the workspace is a Unity project by checking that all of these directories exist:

- `Assets/`
- `Packages/`
- `ProjectSettings/`

If any are missing, stop and report that Unity MCP settings were not written because the workspace is not a Unity project.

## Configuration location

Write Unity MCP configuration only to:

`<unity-project-root>/UserSettings/mcp.json`

Never add Unity MCP to:

- Cursor global MCP settings
- `~/.cursor/`
- `.cursor/mcp.json`
- A parent workspace containing multiple projects
- Any non-Unity repository

Create `UserSettings/` when it does not exist. Preserve unrelated servers and top-level properties in an existing `mcp.json`; add or update only `mcpServers.unity-mcp`.

## Relay configuration

Use a project-local stdio entry:

```json
{
  "enabled": true,
  "path": "",
  "mcpServers": {
    "unity-mcp": {
      "type": "stdio",
      "command": "<absolute-relay-path>",
      "args": ["--mcp"]
    }
  }
}
```

Choose the relay executable for the current operating system:

- Windows: `%USERPROFILE%/.unity/relay/relay_win.exe`
- macOS Apple Silicon: `$HOME/.unity/relay/relay_mac_arm64.app/Contents/MacOS/relay_mac_arm64`
- macOS Intel: `$HOME/.unity/relay/relay_mac_x64.app/Contents/MacOS/relay_mac_x64`
- Linux: `$HOME/.unity/relay/relay_linux`

Resolve environment variables and home-directory shortcuts before saving. Store an absolute path because MCP clients might not expand `~` or `%USERPROFILE%`.

## Workflow

1. Locate the Unity project root, not merely the current subdirectory.
2. Apply the scope guard.
3. Resolve the platform-specific relay to an absolute path.
4. Check whether the relay exists.
5. Read and parse an existing `UserSettings/mcp.json`.
6. Preserve unrelated configuration and upsert `mcpServers.unity-mcp`.
7. Write valid, indented JSON.
8. Read the file back and confirm it parses.
9. Report the project-local path and whether the relay exists.

If the relay is absent, still save the valid project-local configuration, then explain that opening Unity with its MCP/AI Assistant integration should install the relay.
