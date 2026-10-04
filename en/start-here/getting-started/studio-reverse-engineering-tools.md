# Install reverse-engineering tools in Studio

Status: current managed tool workflow, updated 2026-10-05. Availability depends on the installed build.

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

Installed tools start on demand when an assistant session opens or you change a tool's window option.
Each installed tool has separate **Background** and **Visible** options in the tools dialog, saved
for this Studio game. In Visible mode, you and the assistant use the same managed instance and project.
You can inspect the tool window and make manual changes; verify the selected process/program again
before the assistant continues. These options do not connect to a CE or Ghidra window you opened separately.

CE shows or hides its window without restarting, retaining its table and selected process.
Ghidra waits for this game's AI conversations to finish, saves the project, and restarts in the chosen
mode. The dialog shows waiting/changing progress and any original error. A failed save stops the
mode change. CE's visibility remains available while Ghidra waits for an AI run.

CE uses a private instance, hidden by default,
which starts unattached. It can select the game's process or another program you ask to inspect.
Call `instance_list`, then `process_list`, `process_attach` with a positive PID or exact process name,
and `process_get_current` to verify the selection. Pass the discovered `instanceId` to every CE call.

Ghidra uses a persistent project for this Studio game, with Background mode as the default. It starts with no automatic
game import. Use `import_binary` with an absolute executable or DLL path, then `list_project_binaries`
to check readiness and obtain the exact `binary_name` for analysis calls. Ghidra provides static file
analysis here; the managed integration does not expose live debugger attachment.

Ask the assistant to inspect **get_reverse_engineering_status** first. This reports installation,
suggested game executable/PID, target-selection guidance, host availability and original startup errors.
The suggested game PID is not a receipt for CE's current selection. Ghidra analysis/indexing runs
asynchronously. Analysis, code search and string search have separate readiness flags. The assistant
can use `wait_for_binary` with capability `analysis`, `code` or `strings` and a bounded timeout of up
to 60 seconds. Cancelling the wait leaves analysis running. A temporary `not_ready` result does not
mean the binary is missing. Short import names from `list_imports` can be used with `list_xrefs`;
ambiguous names require a qualified DLL name or address.

For example:

> Check the attached game's health with CE, find the code that changes it, and inspect the related
> Ghidra pseudocode and references. Explain the findings before changing memory or creating a package.

Conversations for the same game share managed tools and CE's selected target; coordinate changes
with other running conversations. Other games have separate instances/projects.
Installation or managed-host changes refresh the tool connection on the next message, preserving the
conversation. A message already running retains its current connection. A tool that fails to start
does not stop ordinary package editing. Report its original failure rather than assuming it worked.
An exited managed host restarts at the next message. Starting or restarting a game does not reset a
trainer or other program explicitly selected through CE. If the selected process itself exits,
attach to its new PID and rediscover live addresses before continuing.

Windows permissions still apply. Normal `process_create` reports the original Windows error if
launch fails, and confirms the started PID separately from CE attachment. Error 740 means Windows
requires elevation; it does not mean the trainer started or exited. Debugger launch can report
unverified creation if CE does not select a new PID. Check the actual process list before retrying.
A trainer that requires elevation must be launched with the user's
approval; an ordinary Studio/CE process may be unable to inspect an elevated target. This workflow
does not automatically elevate the app, approve UAC prompts or change security settings.

The app closes its managed tool instances at shutdown and leaves your separate CE/Ghidra instances
alone. Persistent analysis projects and logs are local under
`%LOCALAPPDATA%/Hexinton Mod/studio-tool-data`; installations are under
`%LOCALAPPDATA%/Hexinton Mod/studio-tools`.

Finding a live address and understanding static pseudocode are different steps. Translate static
addresses using the module's RVA and live base, verify the actual process/CE instance, and resolve
heap pointers again after recreation or restart. Tool output is evidence, not proof a mod is correct.
Package changes still follow [the manual Apply and testing workflow](studio-ai-assistant.md).
Tool installation and window changes do not Apply or enable packages.
