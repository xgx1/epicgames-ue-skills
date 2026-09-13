---
name: unreal-mcp
description: "Live-editor MCP actions in Unreal Engine (unreal-mcp): change, query, or run project state. Not for conceptual or docs questions."
---

# Unreal MCP

You are wired into a live Unreal Editor through the `unreal-mcp` MCP server. The server exposes hundreds of tools across 30+ toolsets (actors, blueprints, materials, Niagara, Sequencer, Control Rigs, GAS, automation tests, Live Coding, and more) registered through Unreal's `ToolsetRegistry`. Use it to inspect and mutate live editor state instead of telling the user to do it manually.

You don't need to memorize tool names. The flow below has you discover them on demand.

## First step every time: discover the tool you need, then dispatch it via `call_tool`

Tool search is on by default, so the MCP server advertises only three meta-tools for the whole session: `list_toolsets`, `describe_toolset`, and `call_tool`. Tool names like `BlueprintTools.create` or `SequencerTools.create_level_sequence` are **not in `tools/list`**. They are dispatched server-side through `call_tool` and never registered as native MCP tools. This is deliberate. It keeps your context window small and the prompt cache warm.

When you start work:

1. Call `list_toolsets` to see what's registered, then `describe_toolset` on the candidate(s) to read their tool schemas. If you already know which toolset you need (the user said "make a Blueprint" → the Blueprint toolset), skip the listing and go straight to `describe_toolset` to confirm the available tools and their signatures.
2. Invoke the tool with `call_tool`: pass `toolset_name`, `tool_name`, and an `arguments` object matching the schema you just read. The result comes back on the same turn. No extra round-trip needed.
3. Top-level dispatch (omitting `toolset_name`) is reserved for tools registered directly on the MCP server and is rejected for `call_tool` itself.

If the meta-tools themselves aren't available (`list_toolsets` errors, or you don't see `unreal-mcp` in your MCP server list at all), the editor or its MCP server is not running. Don't bluff. Ask the user to launch the editor (and run `ModelContextProtocol.StartServer` in the console if auto-start isn't on), or follow `references/setup.md` to wire up a project that has never been configured.

## Safety rules

These exist because every MCP call mutates live editor state and runs on the game thread. Treat them as hard constraints, not suggestions.

- **Save first, then save again.** Tell the user to save the project (or call `AssetTools` save APIs) before any bulk change, and again after. MCP edits are not always undoable, especially across compilation boundaries. Treat anything that touches multiple assets as a destructive operation that needs a recovery point.
- **Wait for compilation.** If C++ or shader compilation is in flight, your tool calls will hang or fail in confusing ways. To rebuild C++ from the running editor, drive `LiveCodingToolset.CompileLiveCoding` and wait on its result instead of asking the user to switch to the IDE. That tool blocks until the compile actually finishes and surfaces MSVC diagnostics.
- **Sequential, never parallel.** Tool calls execute on the game thread, so issuing them in parallel deadlocks or fails. Even when calls look independent, serialize them.
- **Always check the result.** Blueprint compilation, widget creation, material edits: many tools return a status that flips between success and failure with no exception thrown on the wire. Read the response before moving on. Treat anything that isn't an explicit success as a stop.
- **Mind PIE.** Editor-only tools (asset creation in particular) behave differently while Play-in-Editor is active. If a result looks wrong, check whether PIE is running and stop it if so.
- **Start PIE in its own floating window.** Every `StartPIE` call uses `playMode: "PlayMode_InEditorFloating"` (2026-09-08 user directive, verified). The standalone PIE window is an independent top-level window (its own entry in `SlateInspector.Windows`, e.g. `… 预览 [NetMode: Standalone 0]`), so visual verification screenshots exactly that one window. Viewport-embedded PIE shares the editor frame, invites modal dialogs, and deforms with layout changes.

## Project skills

A project or plugin can register **Agent Skills**: named bundles of instructions that capture workflow knowledge the agent wouldn't otherwise have (a project's naming conventions, folder layout, required setup steps, or the canonical sequence for a multi-step task). These are separate from the toolsets themselves and are reached through the agent skill toolset (`AgentSkillToolset`), not through `list_toolsets`.

Check for them the same way you discover tools, and do it whenever you start unfamiliar work in a project rather than just once:

1. Call `AgentSkillToolset.ListSkills` (through `call_tool`) to see what skills the project registers. Each entry carries a short description of what it covers and when it applies.
2. If a skill's description looks relevant to what the user asked, call `AgentSkillToolset.GetSkills` on it to load the full instructions, then follow them.
3. If nothing matches, fall back to the tool-discovery flow above.

A relevant project skill's instructions take precedence over your generic defaults: it exists precisely because the project's way of doing something differs from the obvious one. (Authoring or editing these skills is a separate task covered by the `unreal-skill` companion skill below.)

## Reference files

- `references/setup.md`: first-time MCP server setup for a project that has never been configured (`.uproject` plugin entry, auto-start `.ini`, `.mcp.json` generation).
- `references/operations.md`: console commands, settings, and a troubleshooting matrix for when things go wrong (port collision, missing toolsets, hangs, empty docked context).

## Companion skills

- **`create-toolset`**: use when authoring a new toolset or adding tools to an existing one. Covers design principles, C++ and Python conventions, registration, error handling, and testing.
- **`unreal-skill`**: use when creating, updating, or reviewing an Agent Skill. Covers what makes a good skill and how to structure one.

## CLI/curl 直连 HTTP 实战（无 MCP 客户端环境，2026-09-08 实测 UE 5.8）

编辑器常驻场景下可以用 curl 直接打 `http://127.0.0.1:8000/mcp`（Streamable HTTP + JSON-RPC 2.0）：

```bash
SID=""; MCP() { curl -s -m 90 -X POST http://127.0.0.1:8000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  ${SID:+-H "Mcp-Session-Id: $SID"} -D /tmp/hdrs -d "$1"; }

# 1) initialize：响应头里拿 Mcp-Session-Id（后续每次调用都要带）
MCP '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"cli","version":"1.0"}}}'
SID=$(grep -i 'mcp-session-id' /tmp/hdrs | tr -d '\r' | awk '{print $2}')
# 2) 完成握手（notification，无 id）
MCP '{"jsonrpc":"2.0","method":"notifications/initialized"}' > /dev/null
# 3) tools/list 拿三个元工具的 schema；tools/call 调用
MCP '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_toolsets","arguments":{}}}'
```

要点（踩过的坑）：

- **`-D` 落响应头必须写在函数体内的 curl 上**——写成 `MCP '{…}' -D /tmp/hdrs`（函数调用后面）会被包装函数吞掉，拿到空 SID，下一条就报 "Missing required Mcp-Session-Id"。
- **响应是 SSE 帧**（`data: {...}` 行）或纯 JSON，解析前先剥 `data:` 前缀。
- **`call_tool` 内层参数键名是 `arguments`**——写 `params`/`tool_parameters` 会收到 "incoming function input params Json is empty"（内层 schema 为空）。完整形状：`{"name":"call_tool","arguments":{"toolset_name":"…","tool_name":"…","arguments":{…内层 schema…}}}`。
- `describe_toolset` 的参数名是 **`toolset_name`**。
- 参数不匹配时错误信息会**回显内层 schema**——盲调一次就能拿到准确参数名，比翻文档快。
- 打开资产编辑器用 `EditorToolset.EditorAppToolset.OpenEditorForAsset {"assetPath":"/Game/UI/WBP_X"}`（5.8 python 的 `EditorAssetLibrary.open_editor_for_asset` 不存在，但 MCP 工具在）；随后 `Windows {"action":"list"}` 看顶层窗口（资产编辑器是**独立顶层窗口**，不在主框架 tab 里）。截图前必须 `Snapshot {"ref":"","maxDepth":0}` 取窗口 ref，传 `Screenshot {"ref":"wN"}`——空 ref 在编辑器窗上只截到 100×50 子控件（实测 2026-09-08）。
- **`SlateInspectorToolset.SlateInspectorToolset.Screenshot` 是无合成器 Linux 上唯一可靠的视觉截图**（Slate 级截屏，绕开「Vulkan 直通窗口被 X11 抓成纯黑」和「FWidgetRenderer 画零像素」两个死坑）。返回 `returnValue` 是 `{mimeType:"image/png", data:base64}`，base64 解码存 PNG。**ref 传参规则**：PIE 浮动窗口用 `{"ref":""}`（空 ref → 激活窗口，实测 1272×692 完整画面）；编辑器资产窗口（蓝图/UMG/材质）**不可用空 ref**（只截到 100×50 子控件）——必须先 `Snapshot` 取窗口具体 ref（如 `w4`）再 `{"ref":"w4"}`。
- `Snapshot {"ref":"","maxDepth":0}` 返回纯文本 `window "标题" [size=W,H] [ref=wN]`——从中提取目标窗口的 ref，这是编辑器资产窗口截图的**唯一正确输入源**。
- 编辑器退出段错误（139）发生在关闭尾巴，先落盘的成果不受影响。

## UMG 视觉验证标准路径（Linux，2026-09-08 五轮实测定稿 + 六步验证修正）

UMG 截"效果验证图"，以下三条路**全部纯黑**（PNG 逐通道统计 min=max=0 核实）：commandlet + FWidgetRenderer（commandlet 强制 NullRHI）；GUI 编辑器内 FWidgetRenderer（控件状态诊断全对仍画零像素）；X11 抓窗（无合成器时 Vulkan 直通绕过帧缓冲，编辑器 chrome 可见、Vulkan 区域黑）。**唯一正解**：编辑器常驻 + MCP `SlateInspector.Screenshot`。

正确流程（2026-09-08 实测修正——原「Windows list/select → Screenshot ref=""」在资产编辑器窗上只截到 100×50 子控件，不可用）：

1. 启动编辑器 → curl MCP 握手
2. `OpenEditorForAsset {"assetPath":"/Game/UI/WBP_X"}` 打开资产编辑器
3. `Snapshot {"ref":"","maxDepth":0}` → 输出纯文本 `window "WBP_X" [size=W,H] [ref=wN]` → 提取目标窗口的 ref（如 `w4`）
4. `Screenshot {"ref":"w4"}` → 返回 `{mimeType:"image/png", data:base64}` → base64 解码存 PNG
5. 读图核验（布局/控件/文本）

PIE 浮动窗口截图更简单：`Windows {"action":"select","index":N}` 激活 PIE 浮窗 → `Screenshot {"ref":""}`（空 ref 对游戏窗口生效，实测 1272×692 完整画面）。
