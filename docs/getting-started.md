# Getting started: build and use Unity Mod Manager from source

This guide takes you from a fresh clone of this repository to mods running in your game, on Windows. If you only want to use Unity Mod Manager (UMM) and don't need to build it yourself, download the ready-made release from the [Nexus Mods page](https://www.nexusmods.com/site/mods/21) instead, and skip to [Part 4](#part-4-install-the-mod-manager-into-a-game).

## How the pieces fit together

UMM has two parts:

- **The installer app** (`UnityModManager.exe`) runs on your PC. You use it to pick a game, install the mod manager into that game, and add mods.
- **The in-game mod manager** (`UnityModManager.dll`) is copied into each game by the installer app. It loads your mods when the game starts and adds the in-game UMM window.

Building this repository produces both. Each game keeps its own copy of the in-game part, so a rebuild doesn't change a game until you install into it again (see [Updating](#updating)).

## Contents

1. [Install the build tools (one time)](#part-1-install-the-build-tools-one-time)
2. [Build the app](#part-2-build-the-app)
3. [Add the game list](#part-3-add-the-game-list)
4. [Install the mod manager into a game](#part-4-install-the-mod-manager-into-a-game)
5. [Install mods](#part-5-install-mods)
6. [Use UMM in the game](#part-6-use-umm-in-the-game)
7. [Updating](#updating)
8. [Troubleshooting](#troubleshooting)

---

## Part 1: Install the build tools (one time)

You need three things: Visual Studio Build Tools 2022, the .NET 9 SDK, and access to the nuget.org package feed.

### 1.1 Visual Studio Build Tools 2022 with ".NET desktop build tools"

1. Download and install [Visual Studio Build Tools 2022](https://visualstudio.microsoft.com/downloads/) if you don't have it. A full Visual Studio 2022 install also works.
2. Open **Visual Studio Installer** from the Start menu.
3. Next to **Visual Studio Build Tools 2022**, click **Modify**.
4. On the **Workloads** tab, tick **.NET desktop build tools**.
5. In the **Installation details** panel on the right, make sure **.NET SDK** is ticked (it is by default).
6. Click **Modify** and accept the administrator prompt.

To do the same from an administrator PowerShell window:

```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\Installer\setup.exe" modify `
  --installPath "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools" `
  --add Microsoft.VisualStudio.Workload.ManagedDesktopBuildTools --includeRecommended --passive
```

### 1.2 .NET SDK 9.0.3xx

The repository's [`global.json`](../global.json) tells the build to use .NET SDK 9.0.3xx. Newer SDKs (10 and later) need a newer MSBuild than Visual Studio 2022 has, so the build would fail with them.

Check which SDKs you have:

```powershell
dotnet --list-sdks
```

If there's no `9.0.3xx` line, install the latest .NET 9 SDK from <https://dotnet.microsoft.com/download/dotnet/9.0>. Having .NET 10 installed alongside it is fine.

### 1.3 The nuget.org package feed

The build downloads its libraries (Harmony, dnlib, and others) from nuget.org. Check that it's in your package sources:

```powershell
dotnet nuget list source
```

If `nuget.org` isn't listed, add it:

```powershell
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org
```

---

## Part 2: Build the app

From the repository root, run:

```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin\MSBuild.exe" UnityModManager.sln /restore /p:Configuration=Release /m
```

If you installed full Visual Studio instead of Build Tools, replace `BuildTools` in the path with `Community`, `Professional` or `Enterprise`.

The build should end with `Build succeeded` and `0 Error(s)`. It also prints several hundred warnings, mostly about missing code comments. Those are normal and safe to ignore.

> **Don't use `dotnet build`.** The project needs the Visual Studio MSBuild above. `dotnet build` fails with `ResGen.exe not supported on .NET Core MSBuild` or with errors `MSB3822` / `MSB3823` about non-string resources.

### Where the build goes

Everything you need ends up in one folder:

```
UnityModManagerApp\bin\Release\net48\
```

| File | What it is |
|---|---|
| `UnityModManager.exe` | The installer app. This is the one you run. |
| `UnityModManagerApp.exe` | An identical copy under the project's own name. You can ignore it. |
| `UnityModManager.dll` | The in-game mod manager that the app copies into games. |
| `Console.exe` | A text-only version of the installer. |
| `Downloader.exe` | The updater that the app uses to download official releases. |
| `0Harmony.dll`, `dnlib.dll`, `Harmony\`, `winhttp_x64.dll`, `winhttp_x86.dll`, ... | Libraries the app copies into games during installation. |

Keep these files together. The app needs the files next to it to install into games, so if you want UMM somewhere else, copy the **whole folder**, not just the `.exe`.

---

## Part 3: Add the game list

The app needs a file called **`UnityModManagerConfig.xml`** next to `UnityModManager.exe`. It lists every supported game and how to hook into it.

**This file is not in the repository.** It only ships inside the official release zip. Without it, the game list is empty, the **Install** button and **Mods** tab are greyed out, and the log shows:

```
'UnityModManagerConfig.xml' not found.
Error parsing file 'UnityModManagerConfig.xml'.
```

To add it:

1. Download the official release zip, either from the Dropbox link in the [README](../README.md) or from the [Nexus Mods page](https://www.nexusmods.com/site/mods/21).
2. Open the zip, go into the `UnityModManagerInstaller` folder, and copy **only** `UnityModManagerConfig.xml`.
3. Paste it into `UnityModManagerApp\bin\Release\net48\`, next to `UnityModManager.exe`.

Rebuilding doesn't remove the file. Deleting the `bin` folder does, and then you need to copy it in again.

The upstream author adds new games to this file over time. To pick them up, download the release zip again and replace the file.

> **Your game isn't in the list?** You can add it yourself in a separate file whose name starts with `UnityModManagerConfig`, for example `UnityModManagerConfigLocal.xml`. The release zip contains a commented template with that name. See [How to add a new game](https://wiki.nexusmods.com/index.php/How_to_add_new_game_(UMM)) on the Nexus wiki.

### Optional: make the app searchable from the Start menu

Windows Search finds apps through Start menu shortcuts. To add one, run this in PowerShell, changing the path to where your build is:

```powershell
$exe = "D:\path\to\unity-mod-manager\UnityModManagerApp\bin\Release\net48\UnityModManager.exe"
$s = (New-Object -ComObject WScript.Shell).CreateShortcut("$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Unity Mod Manager.lnk")
$s.TargetPath = $exe
$s.WorkingDirectory = Split-Path $exe
$s.Save()
```

Press the Windows key and type "Unity Mod Manager" to find it. To remove the shortcut, delete `Unity Mod Manager.lnk` from `%APPDATA%\Microsoft\Windows\Start Menu\Programs`.

---

## Part 4: Install the mod manager into a game

**Choosing a game in the app does not install anything.** You have to click **Install** for each game you want to mod. This is the step people most often miss.

1. **Close the game.**
2. Run `UnityModManager.exe`. If your game is under `C:\Program Files`, right-click it and choose **Run as administrator** so it can write to the game folder.
3. On the **Install** tab, pick your game from the **Game** list.
4. Check the **Folder** button. If it shows your game's folder name, the app found the game. If it still says **Select**, or the log says `Game folder '...' not found.`, click it and choose the game's folder. That's the folder holding the game's `.exe` and its `<GameName>_Data` folder, for example `D:\Steam\steamapps\common\YourGame`. To type the path instead, use the **...** button next to it.
5. Under **Installation method**, choose one:
   - **DoorstopProxy** (recommended): loads UMM without changing the game's files. It usually keeps working after game updates.
   - **Assembly**: patches one of the game's files. It's more reliable on some games, but you have to install again after every game update.

   If a method is missing or greyed out, it isn't available for that game.
6. Click **Install**.

### How to tell it worked

- The **Install** button changes to **Update** and is greyed out.
- **Ingame Version** shows the version you built, for example `0.33.3`.
- The game folder now contains these new items:

| Location in the game folder | Created by |
|---|---|
| `<GameName>_Data\Managed\UnityModManager\` | Both methods. Contains the in-game manager and, after the game has run once, its settings file `Params.xml`. |
| `Mods\` | Both methods. This is where your mods go. |
| `winhttp.dll` and `doorstop_config.ini` | DoorstopProxy only |

To remove UMM from a game later, select the game and click **Uninstall**. If the game's files were patched, **Restore original files** puts back the backups.

---

## Part 5: Install mods

UMM mods come as `.zip` files, usually from Nexus Mods.

1. In the app, go to the **Mods** tab with your game selected.
2. Click **Install Mod** and choose the mod's `.zip` file. You can also drag the `.zip` onto the mod list.

The app unpacks the mod into the game's `Mods` folder. You can also unzip a mod into that folder by hand.

If the log says `Install the UnityModManager.`, the mod manager isn't installed in this game yet. Go back to [Part 4](#part-4-install-the-mod-manager-into-a-game).

---

## Part 6: Use UMM in the game

Start the game. The UMM window opens by itself at startup. That's controlled by the **Show this window on startup** setting, which is on by default.

| Key | What it does |
|---|---|
| **Ctrl+F10** | Opens or closes the UMM window |
| **Esc** | Closes the UMM window |

The window has three tabs: **Mods** (turn mods on and off and open their settings), **Logs** and **Settings**.

### Change the shortcut

1. Open the UMM window.
2. Go to the **Settings** tab.
3. Next to **Hotkey (default Ctrl+F10)**, click the button showing the current key.
4. Press the key you want, holding Ctrl, Shift or Alt with it if you like.

The shortcut is saved separately for each game.

> **Ctrl+F10 keeps working.** Your shortcut is added alongside Ctrl+F10, not instead of it. The default is built into the code.

### No F10 key on your keyboard?

Try these in order:

1. **Use the window that opens at startup.** It appears without pressing anything, so open its **Settings** tab and set a shortcut you do have.
2. **Fn + 0.** On many compact keyboards, F10 is printed on the `0` key and works with the **Fn** key.
3. **The on-screen keyboard.** Press **Win+Ctrl+O** and click **Fn** to show F1 to F12. It may not reach a game running in full-screen, so switch the game to windowed mode first.
4. **Edit the settings file by hand** (below).

### Edit the settings file by hand

The in-game settings live in:

```
<game folder>\<GameName>_Data\Managed\UnityModManager\Params.xml
```

This file only appears after the game has run once with UMM installed. If the `UnityModManager` folder itself is missing, UMM isn't installed in this game yet (see [Part 4](#part-4-install-the-mod-manager-into-a-game)).

**Close the game before editing.** The game rewrites this file and would undo your change.

Find the `<Hotkey>` block near the top of the file. Edit that one, not the `<Hotkey>` blocks inside `<ModParams>`, which are shortcuts for individual mods. Set the key and modifiers you want:

```xml
<Hotkey>
  <keyCode>Home</keyCode>
  <modifiers>0</modifiers>
</Hotkey>
```

- **`keyCode`** is a Unity key name, such as `Home`, `Insert`, `Pause`, `F9`, `Keypad0` or `BackQuote` (the `~` key). Pick one your game doesn't already use.
- **`modifiers`** says which extra keys to hold. Add the numbers together for combinations:

| Value | Hold |
|---|---|
| `0` | Nothing |
| `1` | Ctrl |
| `2` | Shift |
| `4` | Alt |
| `3` | Ctrl+Shift (1 + 2) |
| `5` | Ctrl+Alt (1 + 4) |

If `Params.xml` doesn't exist yet but the `UnityModManager` folder does, you can create the file with only this content. Every other setting keeps its default.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Param>
  <Hotkey>
    <keyCode>Home</keyCode>
    <modifiers>0</modifiers>
  </Hotkey>
</Param>
```

The same file controls the startup window. If you turned it off and can't open UMM any more, set `<ShowOnStart>1</ShowOnStart>`.

---

## Updating

### After pulling new code

1. Rebuild (see [Part 2](#part-2-build-the-app)).
2. Run the app and select each game. If the new build has a higher version number than the one in the game, click **Update**.

The app only offers **Update** when your build's version is higher than the one installed in the game. If you changed the code without changing the version number, the button stays greyed out. In that case, click **Uninstall** and then **Install**, or raise `<Version>` in [`UnityModManager/UnityModManager.csproj`](../UnityModManager/UnityModManager.csproj).

### Official updates

The app checks for official releases when it starts. If it offers an update and you download it, the official release replaces your own build in that folder.

---

## Troubleshooting

The app shows its messages on the **Log** tab and also writes them to `Log.txt` next to `UnityModManager.exe`.

### Messages in the app

| Message | What it means | Fix |
|---|---|---|
| `'UnityModManagerConfig.xml' not found.` followed by `Error parsing file 'UnityModManagerConfig.xml'.` | The game list is missing. | [Part 3](#part-3-add-the-game-list) |
| `Install the UnityModManager.` | You tried to add a mod, but the mod manager isn't installed in this game, so there's no `Mods` folder yet. | [Part 4](#part-4-install-the-mod-manager-into-a-game) |
| `Game folder '<name>' not found.` | The app couldn't find the game automatically. This is normal for games you don't have installed. | Click the **Folder** button and choose the game's folder. |
| `Select the game folder that contains the 'Data' folder.` | The chosen folder isn't the game's main folder. | Choose the folder that has the game's `.exe` and its `<GameName>_Data` folder. |
| `This game version (IL2CPP) is not supported.` | The game is built in a way UMM can't hook into (it has a `GameAssembly.dll`). | UMM can't mod this game. |

### Build errors

| Error | Cause | Fix |
|---|---|---|
| `NU1100: Unable to resolve '...'` or `No sources found` | NuGet has no package feed configured. | [1.3](#13-the-nugetorg-package-feed) |
| `The SDK 'Microsoft.NET.Sdk' specified could not be found` | The ".NET desktop build tools" workload is missing. | [1.1](#11-visual-studio-build-tools-2022-with-net-desktop-build-tools) |
| An error saying the .NET SDK requires a newer version of MSBuild | No .NET 9.0.3xx SDK is installed, so a newer SDK was picked. | [1.2](#12-net-sdk-903xx) |
| `ResGen.exe not supported on .NET Core MSBuild`, `MSB3822` or `MSB3823` | The build was run with `dotnet build`. | Use the MSBuild command from [Part 2](#part-2-build-the-app). |
