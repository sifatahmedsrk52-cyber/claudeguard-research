# ClaudeGuard: Windows MSIX / Code Integrity Research

Free public documentation of two separate, real Windows bugs that break Claude Desktop on Windows with nearly identical symptoms. This repo exists to document the bugs, with citations, so anyone can verify and fix the problem themselves - the diagnostic/backup tool that automates these checks is a separate paid download, linked at the bottom.

**TL;DR:** if Claude Desktop won't launch on Windows, check which of these two you actually have before you reinstall anything - reinstalling doesn't fix the second one, and reinstalling in general carries a real, confirmed risk of silently wiping Claude Code's local chat history.

## Bug 1: Code Integrity blocking a bundled DLL

**Symptom:** "This app can't open," and the app's MSIX package status shows `Modified, NeedsRemediation`.

**Mechanism:** Recent Claude Desktop MSIX builds on Windows bundle `vk_swiftshader.dll`, a software GPU fallback. On machines with Memory Integrity (HVCI) enabled, Windows Code Integrity blocks that DLL from loading - logged as Event ID 3033 in the CodeIntegrity operational log - and kills the GPU process. Electron tears down the whole app, and the package-servicing layer flags it `Modified, NeedsRemediation`, which survives a normal repair.

**How to check (read-only, changes nothing):**

```powershell
Get-AppxPackage -Name Claude | Select-Object Name, Version, Status
Get-WinEvent -LogName "Microsoft-Windows-CodeIntegrity/Operational" -MaxEvents 20 |
  Select-Object TimeCreated, Id, Message
```

Event 3033 naming `vk_swiftshader.dll` around your crash time confirms this one. Documented with logs, independently reproduced on NVIDIA, AMD, Intel-integrated, and Windows-on-ARM machines, in Anthropic's own bug tracker: [anthropics/claude-code#88323](https://github.com/anthropics/claude-code/issues/88323).

**A second, independent trigger on Copilot+ ARM64 machines specifically:** Windows' own Auto Super Resolution feature injects `SuperResExt.dll` into the same GPU process, and Code Integrity blocks that one the same way - same event ID, same crash pattern, different DLL, different fix (removing the OS component, not touching Claude). Diagnosed in detail at [electron/electron#53859](https://github.com/electron/electron/issues/53859).

## Bug 2: a leaked kernel handle on the app's container

**Symptom:** "Another program is currently using this file," pointing at a path under `C:\Program Files\WindowsApps\Claude_...`. Ending every `claude.exe` process does nothing. No file is actually locked.

**Mechanism:** this one lives a layer deeper, in the AppX/Desktop Bridge container machinery, not Code Integrity. Claude's packaged container gets a deterministic name. If you trigger a UAC elevation prompt from inside Claude, Windows' AppInfo service opens a handle to that container's Job object to process the request - and on exit or update, that handle isn't always released. A Job Object with any open handle stays alive in the kernel even after it's gone from the visible namespace. The next launch tries to create a new container with the same name, collides with the zombie object, and fails with `ERROR_SHARING_VIOLATION` (0x80070020) - which Windows then shows you as a generic, misleading "file in use" dialog.

**How to check (read-only, changes nothing):**

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-AppModel-Runtime/Admin' -MaxEvents 3000 |
  Where-Object { $_.Id -in 208,215 -and $_.Message -match 'Claude_' } |
  Select-Object -First 5 TimeCreated, Id, Message
```

Event 215 ("cannot create the Desktop AppX container... due to an error converting the job") and 208 at your crash time confirm this one instead. You cannot manually restart the AppInfo service to clear it - Windows blocks that on purpose, since AppInfo gates all elevation requests. A full reboot or just signing out and back in works, because both force-kill every process holding the stale handle. Root-caused in detail: [anthropics/claude-code#53247](https://github.com/anthropics/claude-code/issues/53247), [#61635](https://github.com/anthropics/claude-code/issues/61635), [#73107](https://github.com/anthropics/claude-code/issues/73107).

## Why this distinction matters

These two bugs get conflated constantly because the symptom family overlaps, but the fixes don't transfer:
Bug 1 is cleared by repair/reinstall, or permanently sidestepped by switching to the non-MSIX Squirrel build. Bug 2 isn't fixed by reinstalling at all - the container handle leak has nothing to do with which build is installed. Only a reboot or sign-out actually clears it.

Reinstalling to fix Bug 2 doesn't just fail to help - reinstalling in general carries a separate, confirmed risk: it can silently wipe Claude Code's local chat history while leaving other data (regular chat, project files, memory) intact, and the lost sessions are often unrecoverable from the UI even though the transcript files are still sitting on disk. Misdiagnosing which bug you have and reinstalling "just in case" is the one move that can turn an annoying launch failure into actual data loss.

## If you'd rather not run these checks by hand

ClaudeGuard (https://sidheart.gumroad.com/l/claudeguard, $9.99 one-time) automates the Bug 1 / Code Integrity side of this: it runs the diagnostic above automatically, backs up ~/.claude before you touch anything, and verifies nothing was lost after a reinstall. Everything above is free regardless of whether you ever use it - this repo exists to document the bug, not gate the information behind a purchase.
