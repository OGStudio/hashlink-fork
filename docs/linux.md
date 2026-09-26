# HashLink on Linux

How to build HashLink on Linux (x86_64 or aarch64), install it into a
self-contained prefix, and link HL/C programs against that prefix.

**Status:** the `Makefile` and CMake changes described here were reviewed but
not yet run on Linux. Statements that still need a Linux run are marked
*(unverified)*. The GitHub Actions `linux` jobs build with both build systems.

## Prerequisites

A C compiler (GCC or Clang), `make`, `pkg-config`, and on Debian / Ubuntu the
packages the CI installs (`.github/workflows/build.yml`):

    sudo apt-get install --no-install-recommends -y \
      libmbedtls-dev libopenal-dev libpng-dev libturbojpeg-dev \
      libuv1-dev libvorbis-dev libsqlite3-dev

`sdl.hdll` needs SDL3. Distributions that ship `libsdl3-dev` can install it;
others (Ubuntu 24.04 included) have to build SDL3 from source, as the CI does.
First the SDL build dependencies
([SDL's README-linux](https://github.com/libsdl-org/SDL/blob/main/docs/README-linux.md#build-dependencies)):

    sudo apt-get install --no-install-recommends -y \
      libasound2-dev libpulse-dev libaudio-dev libjack-dev libsndio-dev \
      libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxfixes-dev \
      libxi-dev libxss-dev libxtst-dev libxkbcommon-dev libdrm-dev libgbm-dev \
      libgl1-mesa-dev libgles2-mesa-dev libegl1-mesa-dev libdbus-1-dev \
      libibus-1.0-dev libudev-dev libpipewire-0.3-dev libwayland-dev \
      libdecor-0-dev liburing-dev

then SDL itself:

    wget https://github.com/libsdl-org/SDL/releases/download/release-3.4.10/SDL3-3.4.10.tar.gz
    tar xzf SDL3-3.4.10.tar.gz
    cmake -S SDL3-3.4.10 -B SDL3-build
    cmake --build SDL3-build --parallel
    sudo cmake --install SDL3-build

To skip SDL, build without it: `make -o sdl.hdll` with the `Makefile`,
`-DWITH_SDL=OFF` with CMake.

Haxe is needed only to compile `.hl` bytecode or HL/C sources.

## Building and installing with the Makefile

    make -j$(nproc) PREFIX=$T
    make install PREFIX=$T
    make clean

`$T` must be an absolute path. No `sudo` is needed when `$T` is writable.
The resulting layout:

| Path | Contents |
| --- | --- |
| `$T/bin/hl` | the VM |
| `$T/lib/libhl.so` | the runtime |
| `$T/lib/*.hdll` | `fmt sdl ssl openal ui uv mysql sqlite heaps` |
| `$T/include/` | `hl.h`, `hl_ffi.h`, `hlc.h`, `hlc_main.c` |

`INSTALL_BIN_DIR`, `INSTALL_LIB_DIR` and `INSTALL_INCLUDE_DIR` override the
three directories individually.

`make clean` after `make install` removes every build product, so the source
tree is left without untracked or ignored files *(verified on macOS;
unverified on Linux, but the rules are the same)*.

## How hl finds libhl.so and the .hdll files

`hl` is linked with

    -Wl,-rpath,.:$ORIGIN:$ORIGIN/../lib:$(INSTALL_LIB_DIR)

and the linker records it as `DT_RUNPATH` on current distributions. Check it
with

    readelf -d $T/bin/hl | grep -E 'RPATH|RUNPATH'

`libhl.so` is found, in order, through `LD_LIBRARY_PATH`, then the RUNPATH of
`hl`: the current directory, the directory of `hl`, `bin/../lib`, and the
absolute `$(INSTALL_LIB_DIR)`, then `ld.so.cache` and the default
directories. The tree is therefore relocatable as long as `bin/` and `lib/`
stay side by side.

`.hdll` files are loaded by `src/module.c` with `dlopen("<name>64.hdll")`
and then `dlopen("<name>.hdll")`. `module.c` is part of the `hl` executable,
not of `libhl.so`, and for a name without a slash glibc searches:

1. `LD_LIBRARY_PATH`;
2. the `DT_RUNPATH` of the calling object, `hl`: the current directory (the
   `.` entry), the directory of `hl`, `bin/../lib`, `$(INSTALL_LIB_DIR)`;
3. `ld.so.cache`, then `/lib` and `/usr/lib`.

So the same RUNPATH covers both `libhl.so` and the `.hdll` files. Before the
`$ORIGIN/../lib` entry was added, a tree that had been moved away from the
`PREFIX` it was linked for failed with `Failed to load library fmt.hdll`
unless `LD_LIBRARY_PATH` was set.

Each `.hdll` carries the same RUNPATH as `hl`, which lets it find
`libhl.so`; in practice `libhl.so` is already loaded by then.

Setting `HL_DISABLED_LIBS` to a comma separated list of names refuses to load
those libraries.

## Linking an HL/C program against an installed tree

    haxe -hl gen/main.c -main Main
    cc -O2 -o out -I gen -I $T/include gen/main.c \
       -L $T/lib -lhl -l:fmt.hdll -lm -ldl -Wl,-rpath,$T/lib

Add one `-l:<name>.hdll` per entry of `"libs"` in `gen/hlc.json` (other than
`std`). Use `-Wl,-rpath,'$ORIGIN/...'` instead of an absolute path if the
program and the tree move together.

Both `-l:fmt.hdll` and passing `$T/lib/fmt.hdll` by path are supported: the
`.hdll` files and `libhl.so` are linked with `-soname <file name>`, so the
program records a leaf `DT_NEEDED` either way. Check with

    readelf -d out | grep NEEDED    # libhl.so, fmt.hdll, ...

*(unverified on Linux)*

## CMake

    cmake -S . -B <build> -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$T
    cmake --build <build> --parallel
    cmake --install <build>

This gives the same `bin/`, `lib/`, `include/` layout, with these
differences:

- `libhl` is versioned: `libhl.so -> libhl.so.2 -> libhl.so.2.0.0`, and
  programs record `libhl.so.2`. `-lhl` works with both.
- `CMAKE_INSTALL_LIBDIR` comes from `GNUInstallDirs`, which is `lib64` on
  some non-Debian distributions; pass `-DCMAKE_INSTALL_LIBDIR=lib` to match
  the `Makefile`.
- `hl` gets the RUNPATH `$ORIGIN;$ORIGIN/../lib` (no `.` and no absolute
  path), so `.hdll` files are not looked up in the current directory.
  `libhl` and the `.hdll` files get no RUNPATH; the `.hdll` files find
  `libhl` because it is already loaded.
- On aarch64 the AArch64 JIT backend is compiled
  (`CMAKE_SYSTEM_PROCESSOR` matching `aarch64|arm64`).

`-DDOWNLOAD_DEPENDENCIES=ON` fetches and builds SDL3 and openal-soft instead
of using the system packages.

*(CMake on Linux unverified here; the CI `linux` cmake job builds it and runs
`hl --version` from an installed prefix.)*
