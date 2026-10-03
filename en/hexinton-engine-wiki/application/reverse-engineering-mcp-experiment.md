# Reverse-engineering MCP experiment

The experiment below is historical. New builds add [managed Studio installation and hosting](../../start-here/getting-started/studio-reverse-engineering-tools.md);
the qualification below describes the earlier developer setup.

**Status: developer experiment, tested 2026-10-03.** Automatic installation, game attachment and
CE/Ghidra launch controls are not available as Studio product features yet.

Hex Assistant's existing Copilot SDK extension configuration can connect to ordinary MCP
servers. The current experiment uses [CheatEngine.Mcp](https://github.com/CheatEngineNet/CheatEngine.Mcp)
for live memory and [PyGhidra MCP](https://github.com/clearbluejar/pyghidra-mcp) for static analysis.
It retains the existing coding loop and chat tool activity. It does not add a separate coding agent.

## What was tested

Against the native Hexinton Mod Test Game, actual MCP calls:

- imported and analyzed the executable without loading its PDB, located strings/references and
  decompiled its damage handler;
- attached a private CE instance to a private game process, found the player signature, followed
  pointers, and read health and maximum health;
- wrote health, exercised the actual damage command and restored the original value;
- temporarily patched the damage instruction, verified its effect, then released the patch and
  verified both original bytes and behavior;
- reported invalid-memory and missing-instance errors, and cleaned up the owned processes.

A real Hex Assistant conversation also called both servers through Studio's production adapter,
returning live health/max values and static damage findings without editing packages. This proves
that the extension path works; it does not prove a model can create reliable mods for arbitrary games.

## Connection behavior

CE uses a plugin inside CE plus a stdio MCP gateway. A gateway may discover multiple CE instances.
Every routed call uses the exact `instanceId` from discovery. The CE host PID and attached game PID
are separate; restarting CE invalidates its previous instance ID. The gateway does not launch CE
or choose a game automatically.

Ghidra runs headlessly against a persistent analysis project and exposes Streamable HTTP MCP on
a local endpoint. Clients must use the exact binary name returned by `list_project_binaries`.
Ghidra image addresses must be translated using module RVAs before using them in a live process;
heap pointers must be resolved again after restart or player recreation. Decompilation is evidence,
not a substitute for reading instructions and checking behavior.

Preserve original tool failures. Some Ghidra batch results contain per-item errors even when the
MCP envelope's `isError` is false. A successful transport call therefore does not establish successful
analysis. CE failures include a concrete kind, message and hint where available.

## Local prerequisites and limits

The tested setup uses Ghidra 12.1.4, JDK 21, PyGhidra MCP 0.2.7, CE 7.7 x64 and the CE MCP
source revision pinned in the developer experiment. CE's plugin needs the Core, ASP.NET and
Windows Desktop .NET 10 runtimes. Tools are stored locally; missing installations fail explicitly.
There is no automatic cloud download, retry or replay. The first Ghidra semantic index may need
network access to download its embedding model; a disconnected first setup has not been qualified.
The coding provider and remote documentation remain online dependencies.

CE was locally installed with explicit permission because its installer could not be extracted.
It is not a completely portable deployment. The installed CE license also requires a separate
commercial license before commercial use; this experiment establishes no product redistribution
permission. CE binaries are not committed or bundled with Studio.

The next product work is game/session binding, attach/reconnect state, process ownership and
focused analysis skills. Package edits must still follow Studio's existing writable-copy, validation,
unit/integration test and manual Apply workflow.

See [Studio AI Assistant](../../start-here/getting-started/studio-ai-assistant.md) for the current
package workflow.

For value scans, debugger captures, address translation and replacement checks,
see [Live Memory to Static Analysis](live-memory-to-static-analysis.md). The extended
controlled evaluation also exercised equal-value HUD/NPC decoys and corrected an
accidental pointer chain through a C++ runtime object. These checks remain necessary
before turning a one-session address discovery into a reusable package.
