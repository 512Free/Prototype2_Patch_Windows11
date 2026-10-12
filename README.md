[README.md](https://github.com/user-attachments/files/33329784/README.md)
# Prototype2Fix – run PROTOTYPE 2 (Steam, 2025 build) on Windows 10/11

## Symptom
The game starts, shows the logos and crashes a few seconds later (at
"Press Enter"). Event Viewer shows `APPCRASH`, `prototype2engine.dll`,
exception `0xc0000005`, fault offset `0x00567a80`. It happens both through
Steam and when launched directly, regardless of CPU affinity, `Windowed` etc.

## Root cause (found by disassembling `prototype2engine.dll`)
The Steam build from 2025 (`Build R on 08/28`, PE timestamp `0x68B09A3A`,
Steam buildid 19788008) has a broken archive system:

1. `ArchiveSystem::Init` (RVA `0x689320`) mounts the game's `.rcf` archives
   ("cement files") from the list in **`cementfiles.p3d`** – a file that is
   **not shipped in the Steam depot**. Init therefore mounts nothing.
2. The fallback – a cooperative fiber ("Cement Thread") that scans
   `<dir>\*.rcf` – never starts, because nothing in this build sets the
   "scan requested" flag (RVA `0xE19864`). It would also scan `downloads\`,
   not the game folder.
3. With no archives mounted, every load falls back to loose files on disk
   (`art\hud\main.gfx` …) that do not exist. Scaleform's
   `GFxLoader::CreateMovie("main.gfx")` returns NULL and the engine calls a
   virtual method on the NULL pointer → crash at `0x10567A80`.

(The classic "too many CPU cores" crash concerns older builds; it is handled
too, but it is not the cause here.)

## What the mod does
`dbghelp.dll` in the game folder is a *proxy DLL*: the engine imports three
functions from dbghelp and Windows searches the game folder first, so the DLL
loads automatically before the engine initialises. It forwards the three
calls to the real `SysWOW64\dbghelp.dll` and additionally:

- **MountFix** (the actual fix): two one-shot breakpoints (int3 + vectored
  exception handler, no game files are modified):
  - at `ArchiveSystem::Init`: the scan directory in the init struct is changed
    from `downloads` to `.` (the game folder);
  - right after Init (RVA `0x685600`, main thread, before the first HUD load):
    the "scan requested" flag is set and the engine's own
    `ArchiveSystem::Pump()` (RVA `0x688BD0`) is run alternately with
    `UpdateAllDrives()` (RVA `0x6888C0`, what the real-time task does every
    frame to drive file I/O) until the fiber reports idle. All eleven `.rcf`
    archives are then mounted in ~100 ms.
  The fix only activates if the DLL's PE timestamp and the code bytes at the
  patch points match, so a future game update disables it (logged) instead of
  crashing.
- CPU affinity limited to `MaxCores` (default 8) + a hook on
  `GetProcessAffinityMask`.
- Crash log (`Prototype2Fix_crash.log`): exception, registers, stack with
  module offsets and strings, position of every thread.
- Optional I/O tracing (`TraceFiles=1`): logs `CreateFileA`, `FindFirstFileA`,
  `ReadFile` and `OutputDebugString`.

Why not `winmm.dll` as the proxy? Steam's overlay (`GameOverlayRenderer.dll`)
imports winmm itself and is loaded before the game – a winmm proxy in the
game folder would never be used. dbghelp is not a KnownDLL and the overlay
does not touch it.

## Installation – easiest: the patcher
Run **`Prototype2Fix-Patcher.exe`** (classic Win32 program, no dependencies):

- finds the game automatically through Steam's registry key +
  `libraryfolders.vdf` (or via `Browse...` → pick `prototype2.exe`);
- shows the game build (PE timestamp of `prototype2engine.dll`) and whether it
  is supported, and whether the patch is already installed (CRC32 check of
  `dbghelp.dll`);
- **Patch**: backs up any foreign `dbghelp.dll`/ini to `Prototype2Fix_backup\`,
  writes the embedded files, verifies them with CRC32;
- **Uninstall**: restores the backup or removes the files;
- **Check** / **Launch game** (via `steam://rungameid/115320`).

## Installation – manual
Put these two files next to `prototype2.exe` (in the Steam folder
`...\steamapps\common\Prototype 2`):

- `dbghelp.dll`
- `Prototype2Fix.ini`

Launch the game through Steam as usual. `Prototype2Fix.log` in the game folder
shows what happened, e.g. `mountfix: archives mounted after 48 pump
iterations, 94 ms`.

Uninstall: delete `dbghelp.dll`.

## Settings (`Prototype2Fix.ini`)
| Key | Default | Description |
|---|---|---|
| `MountFix` | 1 | Mount the archives at startup (the fix itself). |
| `MaxCores` | 8 | Max logical processors. 64 = no limit. |
| `LimitProcessAffinity` | 1 | Really restrict the process. |
| `FakeAffinityMask` | 1 | Let the engine see at most MaxCores. |
| `CrashLog` | 1 | Write `Prototype2Fix_crash.log` on a crash. |
| `Log` | 1 | Write `Prototype2Fix.log` on every start. |
| `TraceFiles` | 0 | Log file I/O (debugging). |

## Building from source
Requires Visual Studio with the x86 tools. `source\build.cmd` builds
`dbghelp.dll`; `source\patcher\build.cmd` builds the patcher (embeds
`dbghelp.dll` and `Prototype2Fix.ini` as resources). `source\test\run.cmd`
builds a small test host that loads the DLL and checks forwarding + core
limiting.

## Files in the zip
- `Prototype2Fix-Patcher.exe` – the installer/uninstaller. This is all you need
  to run; the DLL and ini are embedded in it.
- `Prototype2Fix-files\` – everything else:
  - `README.md` – this file
  - `dbghelp.dll`, `Prototype2Fix.ini` – for manual installation (copy next to
    `prototype2.exe`)
  - `source\` – all source code (mod, patcher, test host; commented with RVAs
    and engine semantics)
