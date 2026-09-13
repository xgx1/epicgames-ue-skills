# Operations: Console Commands, Settings, Recovery

Read this when something is misbehaving with the MCP server (tools missing, port collision, stale tool registry), or when you need a non-default configuration.

## Console commands

Run these from the Unreal Editor console (`~`).

| Command                                              | Use it for                                                                                               |
|------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| `ModelContextProtocol.StartServer [port]`            | Start the MCP server. Pass a port to override the default (e.g. when 8000 is in use).                    |
| `ModelContextProtocol.StopServer`                    | Stop the server. Useful if the registry is in a bad state and you want a clean restart.                  |
| `ModelContextProtocol.RefreshTools`                  | Re-register every toolset. Run this after the user enables a new toolset plugin.                         |
| `ModelContextProtocol.GenerateClientConfig <client>` | Regenerate the per-client config file. Args: `ClaudeCode`, `Cursor`, `VSCode`, `Gemini`, `Codex`, `All`. |

## Tool-search mode

`tools/list` has two modes, controlled by the `bEnableToolSearch` UPROPERTY on `UModelContextProtocolSettings` (default `true`):

- **`True`** (default): `tools/list` returns only `list_toolsets`, `describe_toolset`, `call_tool`. Toolset tools are dispatched server-side through `call_tool` and stay out of the prompt; the catalog never changes mid-session, so the prompt cache stays warm.
- **`False`**: every toolset tool is registered as a native MCP tool at startup, schemas visible upfront. Used by the hash-mapping commandlet.

Override in the same `.ini` used for `bAutoStartServer`:

```ini
[/Script/ModelContextProtocolEngine.ModelContextProtocolSettings]
bEnableToolSearch=False
```

## Troubleshooting matrix

| Symptom                                               | What to do                                                                                                                                                                                                                                                                                                          |
|-------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `unreal-mcp` not in `/mcp`, or `list_toolsets` errors | The editor isn't running, or the MCP server isn't started. Ask the user to launch the editor and run `ModelContextProtocol.StartServer` from the console (or enable `bAutoStartServer` per `setup.md`). Then check the Output Log for startup errors.                                                               |
| Editor logs "Failed to listen on port"                | Another process holds the default port. Change `ServerPortNumber` in the per-user `EditorPerProjectUserSettings.ini` (see `setup.md`), or pass `-ModelContextProtocolPort=<port>` on the next launch; restart the editor, and re-run `ModelContextProtocol.GenerateClientConfig ClaudeCode` to refresh `.mcp.json`. |
| A toolset you expect (e.g. `NiagaraTools`) is missing | Run `ModelContextProtocol.RefreshTools`. If still missing, the toolset's plugin may not be enabled in the `.uproject`. Check there.                                                                                                                                                                                 |
| Tool calls hang or return errors                      | Editor may be busy compiling, loading a level, or in PIE. Wait and retry. For long compiles, prefer `LiveCodingToolset.CompileLiveCoding`. It returns when the compile actually finishes.                                                                                                                           |
| `AIAssistantToolset.GetDockedContext` returns empty   | The Claude Code tab must be docked inside an asset editor (Blueprint, Material, etc.) to provide docked context. If undocked, that tool has nothing to report.                                                                                                                                                      |
| Sequential tool calls collide                         | Tool calls execute on the game thread. Don't issue them in parallel, even when they look independent. Serialize.                                                                                                                                                                                                    |
| Every call suddenly fails/no response while the editor IS running | The editor was restarted — the HTTP session is gone. Re-run the `initialize` handshake, take the fresh `Mcp-Session-Id` response header, send `notifications/initialized`, and use that header on every later request. Any cached session id (e.g. in a helper script) must be cleared first.                        |
| `Expected non-empty 'name' param for call to tools/call` | `tools/call` top-level `name` must be the meta tool `call_tool`; the real target goes inside the arguments as `toolset_name` + `tool_name` + `arguments`. Don't pass a toolset tool name as the top-level `name`.                                                                                                   |
| `... is not a valid object path for property 'WidgetBlueprint'` (or similar) | Asset object parameters need the FULL object path: `/Game/UI/WBP_ModeSelect.WBP_ModeSelect` (asset path + `.` + object name), not the bare asset path; widgets inside it use `...:WidgetTree.WidgetName`.                                                                                                         |
| PIE: Click tool returns `true` but the UMG button's handler never fires | Direct Slate mouse injection doesn't reach UMG through the game viewport's input routing. Reliable combo: `Click` (to grab focus) then `PressKey Enter`. Same family: `Type` appends (do `Ctrl+A` first to replace), `SelectOption` can't open PIE comboboxes (keyboard: `Down` opens, arrows move highlight, `Enter` commits). Full tested recipe lives in the `browser-unreal-mcp` skill, `references/ue-simulation-decisions.md`. |
| Need to rebuild C++ but no Live Coding channel available | Do NOT run a plain CLI build while the editor is up — UBT silently switches to hot-reload mode, emits `-0001`-suffixed modules, and the next normal editor start fatals with `Trying to recreate changed class '...' outside of hot reload and live coding!`. Stop the editor, delete the suffixed artifacts + hot-reload `.modules`, rebuild, relaunch. Recovery steps live in the `unreal-cmd` skill (section "编译（Editor 目标）与热重载陷阱"). |
