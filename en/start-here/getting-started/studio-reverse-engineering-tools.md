# Install reverse-engineering tools in Studio

Status: available in builds containing the managed tool installer, added 2026-10-03.

Open Studio for a game. If Cheat Engine or Ghidra is missing, the **Reverse engineering tools**
dialog offers installation for each one. You can choose **Continue to Studio** and install later
with the compact tools button. Normal package editing, validation and tests work without these tools.

Each download shows its size, progress and verification phase. **Cancel** stops that installation;
**Retry installation** starts a fresh attempt after a failure. Original download or file-check
errors are displayed. Closing the dialog lets installation continue; closing the app cancels it.
Downloads do not start or retry automatically. Offline mode prevents new downloads; installed tools
remain available locally.

The Windows x64 bundles include their own runtimes:

| Tool | Included dependencies |
| --- | --- |
| Cheat Engine | CE, its MCP plugin/gateway and private .NET Core, Windows Desktop and ASP.NET runtimes |
| Ghidra | Ghidra, Java, Python, MCP dependencies and the semantic-search model |

You do not need to install Python, Java or a .NET SDK separately for these bundles. Archives are
downloaded from Hexinton's public B2 storage and checked before use. The main desktop app's
WebView2 prerequisite is separate. Available bundle versions are pinned to the installed client build.

## Using them with Hex Assistant

Installed tools start on demand when an assistant session opens. CE uses a private hidden instance,
attaching to the game's active process when available. Ghidra runs headlessly and imports the
active game's executable, or the preferred installation executable. If there is no active process,
CE can still discover and attach through its tools; an unknown executable gives Ghidra an empty project.

Ask the assistant to inspect **get_reverse_engineering_status** first. This reports installation,
target executable/PID, host availability and original startup errors. Ghidra analysis/indexing runs
asynchronously; the assistant must check project readiness before interpreting results.

For example:

> Check the attached game's health with CE, find the code that changes it, and inspect the related
> Ghidra pseudocode and references. Explain the findings before changing memory or creating a package.

Conversations for the same game share managed tools; other games have separate instances/projects.
Installation or target changes refresh the tool connection on the next message, preserving the
conversation. A message already running retains its current connection. A tool that fails to start
does not stop ordinary package editing. Report its original failure rather than assuming it worked.
An exited managed host restarts at the next message. A game process restart can interrupt tool calls
in other running conversations; rediscover the current instance and addresses before continuing.

The app closes its managed tool instances at shutdown and leaves your separate CE/Ghidra instances
alone. Persistent analysis projects and logs are local under
`%LOCALAPPDATA%/Hexinton Mod/studio-tool-data`; installations are under
`%LOCALAPPDATA%/Hexinton Mod/studio-tools`.

Finding a live address and understanding static pseudocode are different steps. Translate static
addresses using the module's RVA and live base, verify the actual process/CE instance, and resolve
heap pointers again after recreation or restart. Tool output is evidence, not proof a mod is correct.
Package changes still follow [the manual Apply and testing workflow](studio-ai-assistant.md).
Tool installation does not Apply or enable packages. Headless hosting is the current managed mode;
a GUI/headless settings toggle is not part of this installer release.
