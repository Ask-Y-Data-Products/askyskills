---
name: setup-dev-env
description: "Set up the full Asky development environment on Windows. Opens Notepad++, Cursor editors, Visual Studio instances, ngrok, npm dev server, and pgAdmin. Arranges windows across 3 monitors. Triggers on: setup dev env, start dev environment, open dev env, dev setup."
---

# Asky Dev Environment Setup

Launches and arranges the full development environment across 3 monitors (left, center, right).

## IMPORTANT NOTES

- Always use **Windows paths** (`C:\`) and **Windows commands**
- ngrok must be called with full path: `c:\tools\ngrok.exe`
- npm start (ng serve) will prompt about autocompletion — pipe `echo N |` to auto-decline
- Visual Studio startup projects cannot be reliably automated via COM — open both instances and **tell the user to set startup projects and run manually**
- Use `cmd /k` with `title` to name windows for later identification
- All window placement uses Win32 `SetWindowPos` via PowerShell
- **NEVER use full-screen (SW_SHOWMAXIMIZED with no taskbar)**. Always use `ShowWindow(hwnd, 3)` (SW_MAXIMIZE) which maximizes but keeps the Windows taskbar visible. For the 2x2 grid, use `SetWindowPos` with the screen's **WorkingArea** (not Bounds) so windows don't overlap the taskbar.

## SCREEN LAYOUT

The user has 3 monitors. Determine screen geometry dynamically with:
```powershell
Add-Type -AssemblyName System.Windows.Forms
[System.Windows.Forms.Screen]::AllScreens | ForEach-Object {
    Write-Host "$($_.DeviceName) WorkingArea=$($_.WorkingArea) Primary=$($_.Primary)"
}
```
**Always use `WorkingArea`** (not `Bounds`) for window placement — `WorkingArea` excludes the taskbar.
- **Left screen**: leftmost by X coordinate
- **Center screen**: the primary screen (or middle by X)
- **Right screen**: rightmost by X coordinate

## STEP 1: Open Notepad++ (LEFT screen, maximized)

```powershell
Start-Process "C:\Program Files\Notepad++\notepad++.exe"
```
Wait 2 seconds, then find the Notepad++ window, move it to the left screen, and call `ShowWindow(hwnd, 3)` (SW_MAXIMIZE) so it fills the screen but the taskbar stays visible.

## STEP 2: Open 3 Cursor instances

Launch each with `cmd /c start`:
```powershell
Start-Process cmd -ArgumentList '/c', 'start "" "C:\Users\AvigadOron\AppData\Local\Programs\cursor\Cursor.exe" "C:\work\asky\prism\prismfront"'
Start-Process cmd -ArgumentList '/c', 'start "" "C:\Users\AvigadOron\AppData\Local\Programs\cursor\Cursor.exe" "C:\work\asky\jam\jamback"'
Start-Process cmd -ArgumentList '/c', 'start "" "C:\Users\AvigadOron\AppData\Local\Programs\cursor\Cursor.exe" "C:\work\asky-workspaces\asky-lib"'
```

Wait for windows to appear, then place:
- **prismfront** Cursor → center screen (open this one first)
- **jamback** Cursor → center screen
- **asky-lib** Cursor → right screen

Match windows by checking window titles for the folder names.

## STEP 3: Open 2 Visual Studio instances

```powershell
$devenv = "C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\devenv.exe"
$sln = "C:\work\asky\jam\jamback\JamApi.sln"

# VS #1 — center screen (user will set JamApi.Api as startup and run)
Start-Process $devenv -ArgumentList "`"$sln`""

# Wait 15 seconds for VS #1 to load before opening VS #2
Start-Sleep -Seconds 15

# VS #2 — right screen (user will set asky.core as startup and run)
Start-Process $devenv -ArgumentList "`"$sln`""
```

Wait for both to appear (look for windows with "JamApi" and "Microsoft Visual Studio" in title), then place:
- **VS #1** → center screen
- **VS #2** → right screen

**Tell the user:**
> "Two Visual Studio instances are open. Please:
> 1. In the CENTER screen VS: set **JamApi.Api** as startup project and run (F5)
> 2. Wait for it to start, then in the RIGHT screen VS: set **asky.core** as startup project and run (F5)"

## STEP 4: Open ngrok command prompt

```powershell
Start-Process cmd -ArgumentList '/k', 'title NGROK_CMD & cd /d c:\tools & c:\tools\ngrok.exe http --url=askya.ngrok.io 5141'
```

## STEP 5: Open npm dev server

```powershell
Start-Process cmd -ArgumentList '/k', 'title NPM_DEV & cd /d C:\work\asky\prism\prismfront & echo N | npm start'
```

## STEP 6: Open pgAdmin 4 (LEFT screen)

```powershell
Start-Process "C:\Program Files\pgAdmin 4\runtime\pgAdmin4.exe"
```
Wait for pgAdmin window to appear, then move it to the left screen.

## STEP 7: Arrange the 4 command windows on LEFT screen (quarters)

After VS is running, there will be 4 command/console windows:
1. **NGROK_CMD** — find by title containing "NGROK_CMD"
2. **NPM_DEV** — find by title containing "NPM_DEV" or "ng serve"
3. **dotnet.exe** (x2) — find by title containing "dotnet.exe" (these are the VS debug console outputs)

**Note:** The 2 dotnet.exe windows will only exist after the user manually runs both VS projects. Tell the user you will arrange these windows after they confirm both VS projects are running.

Place all 4 in a 2x2 grid on the left screen:
```
Left screen quarters:
+----------+----------+
| Window 1 | Window 2 |
+----------+----------+
| Window 3 | Window 4 |
+----------+----------+
```

Use `SetWindowPos` to position each:
```powershell
$leftScreen = ... # get left screen bounds
$halfW = $leftScreen.Width / 2
$halfH = $leftScreen.Height / 2
$x = $leftScreen.X
$y = $leftScreen.Y

# Top-left
SetWindowPos($hwnd1, 0, $x, $y, $halfW, $halfH, 0x0040)
# Top-right
SetWindowPos($hwnd2, 0, $x + $halfW, $y, $halfW, $halfH, 0x0040)
# Bottom-left
SetWindowPos($hwnd3, 0, $x, $y + $halfH, $halfW, $halfH, 0x0040)
# Bottom-right
SetWindowPos($hwnd4, 0, $x + $halfW, $y + $halfH, $halfW, $halfH, 0x0040)
```

## EXECUTION ORDER

1. Detect screen geometry
2. Launch Notepad++ → place on left screen maximized
3. Launch 3 Cursor instances → place prismfront & jamback on center, asky-lib on right
4. Launch VS #1 → wait 15s → launch VS #2 → place center/right
5. Launch ngrok cmd
6. Launch npm dev server cmd
7. Launch pgAdmin → place on left screen
8. Tell user to set VS startup projects and run both
9. After user confirms both VS projects are running, arrange the 4 cmd windows in quarters on left screen
