---
name: build-3dmm
description: Build and run Microsoft 3D Movie Maker from source on modern Windows. Use this skill when asked to build, compile, or run 3D Movie Maker (3DMM).
---

# Build & Run 3D Movie Maker on Modern Windows

The original Microsoft repo (microsoft/Microsoft-3D-Movie-Maker) **cannot build** with modern tools — it requires Visual C++ 2.0 from 1994. Use the community fork **3DMMForever** (https://github.com/foone/3DMMForever) which has been modernized for CMake + Visual Studio 2022+.

## Prerequisites

Install these if not already present (use `winget` or VS Installer):

- **Visual Studio 2022+** with the C++ desktop workload (`Microsoft.VisualStudio.Component.VC.Tools.x86.x64`)
- **Windows SDK** (10.0.22621 or later)
- **CMake** 3.23+ (`winget install Kitware.CMake`)
- **Ninja** build system (`winget install Ninja-build.Ninja`)
- **gsudo** for elevation (`winget install gerardog.gsudo`)

## Step 1: Clone the repo to a short path

```powershell
git clone https://github.com/foone/3DMMForever.git C:\3d
```

Use a short path like `C:\3d` — the project has long relative paths that can hit Windows MAX_PATH.

## Step 2: Set up the build environment

You must configure PATH, INCLUDE, and LIB for the MSVC x86 cross-compiler. Find the actual version numbers on the system:

```powershell
# Find MSVC tools version
$vsBase = (Get-ChildItem "C:\Program Files\Microsoft Visual Studio" -Directory | Sort-Object Name -Descending | Select-Object -First 1).FullName
$vsEdition = (Get-ChildItem $vsBase -Directory | Where-Object { $_.Name -match 'Enterprise|Professional|Community|BuildTools' } | Select-Object -First 1).Name
$msvcVersion = (Get-ChildItem "$vsBase\$vsEdition\VC\Tools\MSVC" -Directory | Sort-Object Name -Descending | Select-Object -First 1).Name
$msvcBase = "$vsBase\$vsEdition\VC\Tools\MSVC\$msvcVersion"

# Find Windows SDK version
$sdkVersion = (Get-ChildItem "C:\Program Files (x86)\Windows Kits\10\Include" -Directory | Sort-Object Name -Descending | Select-Object -First 1).Name

$sdkBin = "C:\Program Files (x86)\Windows Kits\10\bin\$sdkVersion"
$sdkInclude = "C:\Program Files (x86)\Windows Kits\10\Include\$sdkVersion"
$sdkLib = "C:\Program Files (x86)\Windows Kits\10\Lib\$sdkVersion"

$env:PATH = "$msvcBase\bin\Hostx64\x86;$sdkBin\x64;$sdkBin\x86;C:\Program Files\CMake\bin;$env:PATH"
$env:INCLUDE = "$msvcBase\include;$sdkInclude\ucrt;$sdkInclude\um;$sdkInclude\shared"
$env:LIB = "$msvcBase\lib\x86;$sdkLib\ucrt\x86;$sdkLib\um\x86"
```

## Step 3: Fix the SDK header conflict

Modern Windows SDK (10.0.26100+) has a conflict between `winnt.h` and `xmmintrin.h` over `_mm_prefetch`. Fix by removing the `#include <iostream>` from `kauai\src\appbwin.cpp`:

**In `kauai\src\appbwin.cpp`:** Remove or comment out `#include <iostream>` and replace the `std::cout.clear()` / `std::clog.clear()` / `std::cerr.clear()` / `std::cin.clear()` calls (in the `_FInitOS` console setup block) with a comment like `// iostream removed to avoid SDK header conflict`.

> **Note:** If your Windows SDK version is older (e.g., 10.0.22621) this conflict may not exist and you can skip this step.

## Step 4: Build

```powershell
cd C:\3d
cmake --preset x86:msvc:release
cmake --build build --config Release
```

All 132 targets should succeed. The output binary is `C:\3d\build\3dmovie.exe`.

## Step 5: Set up the runtime directory

The app needs a specific directory layout and registry entries.

```powershell
# Create run directory
New-Item -ItemType Directory -Path "C:\3d\run\Microsoft Kids\3D Movie Maker" -Force
New-Item -ItemType Directory -Path "C:\3d\run\Microsoft Kids\Users\Melanie" -Force

# Copy the executable
Copy-Item "C:\3d\build\3dmovie.exe" "C:\3d\run\" -Force

# Copy chunk files (the game's data)
Copy-Item "C:\3d\build\chomp\studio\*.chk" "C:\3d\run\Microsoft Kids\3D Movie Maker\" -Force

# CRITICAL: utest.chk must be renamed to 3dmovie.chk — it's the main resource file
Copy-Item "C:\3d\run\Microsoft Kids\3D Movie Maker\utest.chk" "C:\3d\run\Microsoft Kids\3D Movie Maker\3DMovie.chk" -Force

# Copy content files (themes, content packs, etc.)
Copy-Item "C:\3d\cd3\*" "C:\3d\run\Microsoft Kids\3D Movie Maker\" -Recurse -Force -ErrorAction SilentlyContinue
Copy-Item "C:\3d\cd9\*" "C:\3d\run\Microsoft Kids\3D Movie Maker\" -Recurse -Force -ErrorAction SilentlyContinue
```

## Step 6: Set registry entries

The app is 32-bit, so on 64-bit Windows use `WOW6432Node`. Use gsudo for HKLM access:

```powershell
# Create a batch file for registry commands (avoids quoting issues with gsudo)
@"
reg add "HKLM\Software\WOW6432Node\Microsoft\Microsoft Kids\3D Movie Maker" /v InstallDirectory /t REG_SZ /d "C:\3d\run\Microsoft Kids" /f
reg add "HKLM\Software\WOW6432Node\Microsoft\Microsoft Kids\3D Movie Maker" /v HomeDirectory /t REG_SZ /d "C:\3d\run\Microsoft Kids\Users" /f
reg add "HKLM\Software\WOW6432Node\Microsoft\Microsoft Kids\3D Movie Maker\Products" /f
"@ | Set-Content "C:\3d\setreg.bat"

gsudo cmd /c "C:\3d\setreg.bat"
```

**Important:** Do NOT include a trailing backslash in registry path values — it escapes the closing quote and corrupts the value.

## Step 7: Launch

```powershell
Start-Process -FilePath "C:\3d\run\3dmovie.exe" -WorkingDirectory "C:\3d\run"
```

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `_FFindMsKidsDir` | Registry `InstallDirectory` wrong or missing | Verify it points to `C:\3d\run\Microsoft Kids` (no trailing backslash) |
| `_FInitProductNames:_FFindProductDir` | `3DMovie.chk` missing in product dir | Copy `utest.chk` as `3DMovie.chk` into `Microsoft Kids\3D Movie Maker\` |
| `_FReadStringTables` | Wrong file used as `3DMovie.chk` | Must be `utest.chk` renamed, NOT `studio.chk` — utest.chk contains the string tables |
| Debug assertion failures | Built in Debug mode | Rebuild with `x86:msvc:release` preset |
| `_mm_prefetch` C2733 error | SDK/MSVC header conflict | Remove `#include <iostream>` from `appbwin.cpp` |
| `rc.exe` not found | Windows SDK bin not in PATH | Add `$sdkBin\x64` and `$sdkBin\x86` to PATH |
| `MSVCRTD.lib` not found | LIB path incorrect | Ensure LIB points to `MSVC\...\lib\x86` and SDK `ucrt\x86` + `um\x86` |

## Key Technical Details

- **Target architecture:** x86 (32-bit) — cross-compiled from x64 host
- **CMake presets:** `x86:msvc:debug`, `x86:msvc:release`, `x86:msvc:relwithdebinfo`, `x86:msvc:minsizerel`
- **Registry path (64-bit OS):** `HKLM\Software\WOW6432Node\Microsoft\Microsoft Kids\3D Movie Maker`
- **Product dir detection:** `utest.cpp` → `_FFindMsKidsDir()` reads registry, then `_FFindProductDir()` looks for `3D Movie Maker` or `3DMovie` subdirectory with a matching `.chk` file
- **The app may still need original CD content** (audio, video assets) for full functionality — those assets are not in the repo
