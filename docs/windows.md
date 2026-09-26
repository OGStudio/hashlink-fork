# HashLink on Windows

How to build HashLink on Windows x64 with CMake and Visual Studio 2022,
install it as a flat, self-contained directory, and link HL/C programs
against it with MSVC.

**Status:** the CMake changes described here were reviewed but not yet run on
Windows. Statements that still need a Windows run are marked *(unverified)*.
`hl.sln` (MSBuild) and the CI's CMake recipe, which unpacks the prebuilt
packages into `include/`, are unchanged.

## Prerequisites

- Visual Studio 2022 with the **Desktop development with C++** workload
  (MSVC v143 and a Windows 10/11 SDK).
- CMake 3.14 or newer on `PATH`. The one Visual Studio installs under
  `Common7\IDE\CommonExtensions\Microsoft\CMake` works.
- Two prebuilt packages, the versions the CI uses:
  - [`SDL3-devel-3.4.10-VC.zip`](https://www.libsdl.org/release/SDL3-devel-3.4.10-VC.zip)
  - [`openal-soft-1.23.1-bin.zip`](https://github.com/kcat/openal-soft/releases/download/1.23.1/openal-soft-1.23.1-bin.zip)

Unpack both into one directory, `<deps>`, and rename each extracted root
folder so that the layout is:

    <deps>\sdl\cmake\SDL3Config.cmake      (and include\, lib\ from the zip)
    <deps>\openal\include\AL\al.h
    <deps>\openal\libs\Win64\OpenAL32.lib
    <deps>\openal\bin\Win64\soft_oal.dll

`video.hdll` would additionally need an ffmpeg SDK as `<deps>\ffmpeg`
(`include\`, `lib\`); the recipe below turns it off.

mbedTLS, libuv, zlib, libpng, libjpeg-turbo, vorbis, SQLite and PCRE2 are
vendored under `include/` and built from source.

## Where the prebuilt packages are looked up

`HL_PREBUILT_DEPS_DIR` (a CMake cache path) names the directory holding
`sdl\`, `openal\` and `ffmpeg\`. It defaults to the source tree's `include\`,
which is where the CI unpacks them, so the CI recipe needs no change.

Individual paths can still be overridden:

| Variable | Default |
| --- | --- |
| `SDL3_DIR` | `${HL_PREBUILT_DEPS_DIR}/sdl/cmake` |
| `OPENAL_LIBRARY` | found in `${HL_PREBUILT_DEPS_DIR}/openal/libs/Win64` |
| `OPENAL_INCLUDE_DIR` | `${HL_PREBUILT_DEPS_DIR}/openal/include` |

The OpenAL runtime is always installed from
`${HL_PREBUILT_DEPS_DIR}/openal/bin/Win64/soft_oal.dll`, renamed to
`OpenAL32.dll`; `SDL3.dll` is installed from the location the SDL3 package
reports for `SDL3::SDL3-shared`.

## Building and installing

From a Developer PowerShell or any shell with `cmake` on `PATH`, with
`<build>` and `<T>` absolute paths outside the source tree:

    cmake -S <hashlink> -B <build> -G "Visual Studio 17 2022" -A x64 ^
      -DCMAKE_INSTALL_PREFIX=<T>/bin ^
      -DCMAKE_INSTALL_INCLUDEDIR=../include ^
      -DFLAT_INSTALL_TREE=ON -DBUILD_TESTING=OFF ^
      -DWITH_VIDEO=OFF -DWITH_DIRECTX=OFF -DWITH_DX12=OFF ^
      -DHL_PREBUILT_DEPS_DIR=<deps>
    cmake --build <build> --config Release --parallel
    cmake --install <build> --config Release

`CMAKE_BUILD_TYPE` is ignored by the Visual Studio generator; `--config`
selects the configuration. `FLAT_INSTALL_TREE` is already the default with
MSVC; it sets the bin and lib directories to the prefix itself.

Nothing is written inside the source tree.

## Resulting layout *(unverified)*

    <T>\bin\hl.exe
    <T>\bin\libhl.dll    <T>\bin\libhl.lib
    <T>\bin\<name>.hdll  <T>\bin\<name>.lib     for fmt sdl openal ssl sqlite ui uv heaps
    <T>\bin\SDL3.dll
    <T>\bin\OpenAL32.dll
    <T>\include\hl.h hl_ffi.h hlc.h hlc_main.c

Every `.hdll` target is installed with a plain `install(TARGETS ...
DESTINATION ...)`, which covers the `RUNTIME` artifact (the `.hdll`) and the
`ARCHIVE` artifact (its MSVC import library). The import library of
`<name>.hdll` is `<name>.lib`, because the targets set `PREFIX ""` and
`OUTPUT_NAME <name>`; the one of `libhl.dll` is `libhl.lib`. If the install
rule is ever split into `RUNTIME` / `LIBRARY` keywords, `ARCHIVE
DESTINATION` must stay, or HL/C programs cannot link.

`mysql.hdll` is built by `hl.sln` but not by CMake on Windows unless
`-DWITH_MYSQL=ON` is passed.

## How hl.exe finds libhl.dll and the .hdll files

`libhl.dll` is an import of `hl.exe` and is found by the standard DLL search,
which starts with the directory of the executable. `.hdll` files are loaded
by `src/module.c` with `LoadLibraryA("<name>64.hdll")` and then
`LoadLibraryA("<name>.hdll")`, which uses the standard search order:

1. the directory of `hl.exe`;
2. the system directories (`System32`, `System`, the Windows directory);
3. the current directory;
4. the directories on `PATH`.

`SDL3.dll` and `OpenAL32.dll` are dependencies of `sdl.hdll` and
`openal.hdll` and are found the same way, which is why they are installed
next to `hl.exe`. The flat `bin\` therefore runs on its own, from any working
directory, and can be moved as a whole.

## Linking an HL/C program with MSVC *(unverified)*

    haxe -hl gen/main.c -main Main
    cl /nologo /O2 /bigobj /I gen /I <T>\include gen\main.c /Fe:out.exe ^
       /link /LIBPATH:<T>\bin libhl.lib fmt.lib

Link one `<name>.lib` per entry of `"libs"` in `gen/hlc.json` (other than
`std`). `hlc_main.c` provides `wmain`, so no `/ENTRY` is needed for a console
program. To run `out.exe`, put `libhl.dll` and the `.hdll` files it links
next to it, or put `<T>\bin` on `PATH`.

`gen\main.c` `#include`s every other generated file, so the whole program is
one translation unit. A translation unit of a few megabytes can exceed the
default COFF section limit of 65,279 and fail with `fatal error C1128`;
`/bigobj` raises the limit and has no runtime cost, so pass it
unconditionally.
