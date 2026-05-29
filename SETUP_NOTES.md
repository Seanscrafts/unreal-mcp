# UnrealMCP — Setup Notes for UE 5.5+
**Machine:** Windows 11 — Esme's workstation (Cape Town)
**Unreal project:** BlendMasterMaterial (UE 5.7)
**Date set up:** 2026-05-29

These notes document everything needed to get UnrealMCP compiling and running on Unreal Engine 5.5 or later. The original repo does not compile on UE 5.5+ without the fixes listed below.

---

## What you need installed first

| Requirement | Notes |
|---|---|
| Unreal Engine 5.5 or later | Tested on 5.7 |
| Visual Studio Build Tools 2022 | Install from https://visualstudio.microsoft.com/downloads/ — scroll down to "Tools for Visual Studio", pick "Build Tools for Visual Studio 2022". Select the **Desktop development with C++** workload. |
| .NET Framework 4.6.2 Developer Pack | `winget install --id Microsoft.DotNet.Framework.DeveloperPack.4.6 --silent --accept-package-agreements --accept-source-agreements` |
| Python 3.12+ | Required for the MCP server |
| `uv` | Python package manager used by the MCP server — install from https://docs.astral.sh/uv/ |

**Important:** After installing VS Build Tools, Windows requires a **restart** before it will compile anything. Do the restart before opening Unreal.

---

## Compile fixes for UE 5.5+

The original plugin uses two APIs that were removed or changed in Unreal Engine 5.5. Without these fixes the plugin will not compile.

### Fix 1 — `ANY_PACKAGE` removed (9 occurrences)

`FindObject<UClass>(ANY_PACKAGE, ...)` was removed in UE 5.5.

**Replace every instance with:**
```cpp
FindFirstObject<UClass>(*ClassName, EFindFirstObjectOptions::None)
```

Affected files:
- `Source/UnrealMCP/Private/Commands/UnrealMCPBlueprintCommands.cpp` (4 occurrences)
- `Source/UnrealMCP/Private/Commands/UnrealMCPBlueprintNodeCommands.cpp` (5 occurrences)

### Fix 2 — `TCHAR_TO_UTF8` cast unsafe (2 occurrences)

Direct cast `(uint8*)TCHAR_TO_UTF8(*Response)` causes a compile error in UE 5.5+.

**Replace with:**
```cpp
FTCHARToUTF8 Utf8Response(*Response);
if (!ClientSocket->Send((const uint8*)Utf8Response.Get(), Utf8Response.Length(), BytesSent))
```

Affected file: `Source/UnrealMCP/Private/MCPServerRunnable.cpp` (2 occurrences — lines ~91 and ~315)

### Fix 3 — `BufferSize` name conflict

`const int32 BufferSize = 8192;` conflicts with a name already defined elsewhere in UE 5.5+.

**Rename to:**
```cpp
const int32 RecvBufferSize = 8192;
```

Affected file: `Source/UnrealMCP/Private/MCPServerRunnable.cpp`

---

## Copying the plugin to your Unreal project

1. Copy the entire `MCPGameProject/Plugins/UnrealMCP` folder into your project's `Plugins/` folder
2. Right-click your `.uproject` file → **Generate Visual Studio project files**
3. Open the `.sln` in Visual Studio
4. Build with **Development Editor** target
5. Open your project in Unreal — it will ask to rebuild the plugin — say **Yes**

---

## MCP server setup

The Python MCP server lives in `Python/unreal_mcp_server.py`. It connects to the Unreal plugin over TCP on port **55557**.

### Add to your MCP config

For Claude Code, add this to `C:\ai\.mcp.json` (or wherever your `.mcp.json` lives):

```json
{
  "mcpServers": {
    "unrealMCP": {
      "command": "uv",
      "args": [
        "--directory",
        "C:/ai/unreal-mcp/Python",
        "run",
        "unreal_mcp_server.py"
      ]
    }
  }
}
```

Adjust the `--directory` path if you cloned the repo somewhere else.

### Enable in Claude Code project settings

Add to `.claude/settings.local.json` in your project:

```json
{
  "mcpServers": {
    "unrealMCP": {}
  }
}
```

---

## How to use it

1. Open your Unreal project — the plugin starts a TCP server automatically on port 55557
2. The MCP server (started by Claude Code via the config above) connects to it
3. You can then ask Claude to do things in Unreal in plain language — create actors, set up blueprints, etc.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Plugin won't compile | Check VS Build Tools 2022 is installed with the C++ workload. Restart after installing. |
| MCP server can't connect | Make sure Unreal is open and the project with the plugin is loaded. The plugin only runs while the editor is open. |
| `uv` not found | Install uv: `winget install --id=astral-sh.uv` |
