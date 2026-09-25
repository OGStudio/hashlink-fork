# HashLink on macOS arm64 (Apple Silicon)

Native arm64 build: `hl`, `libhl.dylib` and every `.hdll` are built for
`arm64` and run without Rosetta. The JIT uses the AArch64 backend
(`src/jit_aarch64.c`, `src/jit_aarch64_emit.c`).

Everything below was verified on macOS 26.6.2 (build 25G83), Apple clang
21.0.0, Haxe 4.3.7, with Homebrew under `/opt/homebrew`. Where a statement
was not verified, it says so.

## Prerequisites

Xcode Command Line Tools:

    xcode-select --install

Homebrew packages. `Brewfile` in the repository root lists them, so
`brew bundle` installs the lot; the exact formula names are:

| Formula | Used by | Version verified against |
| --- | --- | --- |
| `jpeg-turbo` | fmt.hdll | 3.2.0 |
| `libpng` | fmt.hdll | 1.6.58 |
| `libogg` | fmt.hdll | 1.3.6 |
| `libvorbis` | fmt.hdll | 1.3.7 |
| `sdl3` | sdl.hdll | 3.4.12 |
| `openal-soft` | openal.hdll | 1.25.2 |
| `mbedtls@3` | ssl.hdll | 3.6.7 |
| `libuv` | uv.hdll | 1.52.1 |
| `sqlite` | sqlite.hdll | 3.53.3 |

Use `mbedtls@3`, not `mbedtls`. The unversioned formula is now mbedTLS 4.x,
while `libs/ssl/ssl.c` targets the 3.x API. Both formulae are keg-only, so
their headers are not under `$(brew --prefix)/include`; the `Makefile`
resolves the keg itself and prefers `mbedtls@3`:

    BREW_MBEDTLS_PREFIX := $(shell brew --prefix mbedtls@3 2>/dev/null)

Without that, `make ssl.hdll` fails with

    libs/ssl/ssl.c:25:10: fatal error: 'mbedtls/error.h' file not found

`sdl3`, `openal-soft` and `sqlite` are keg-only too, and the `Makefile`
already resolves those kegs.

Haxe is needed only to compile `.hl` bytecode or HL/C sources, not to build
the VM.

## Building

From the repository root:

    make -j8

That builds `libhl.dylib`, `hl` and all nine `.hdll` files. Individual
targets work as well:

    make libhl.dylib
    make hl
    make libs          # every .hdll
    make ssl.hdll      # one .hdll

`ARCH` defaults to `$(uname -m)` and is passed to the compiler as
`-arch $(ARCH)`, so a plain `make` on an Apple Silicon Mac produces arm64
binaries. To build for the other architecture, set it explicitly
(`make ARCH=x86_64`); note that Homebrew under `/opt/homebrew` only ships
arm64 libraries, so the `.hdll` files will not link that way.

Set `OSX_SDK` to build against a specific SDK, e.g. `make OSX_SDK=15`.

`make -j8` is clean apart from warnings that are not architecture specific:
unused static functions, `-Wpointer-sign` in `libs/sdl` and `libs/openal`,
`-Wmacro-redefined` for `MSG_NOSIGNAL`, and a few
`-Wincompatible-pointer-types-discards-qualifiers` in `libs/sdl`. There are
no pointer-truncation or integer-conversion warnings.

## Installing

    sudo make install

The install locations come from these variables, all derived from `PREFIX`
(default `/usr/local`):

| Variable | Default | Contents |
| --- | --- | --- |
| `PREFIX` | `/usr/local` | base for the three below |
| `INSTALL_BIN_DIR` | `$(PREFIX)/bin` | `hl` |
| `INSTALL_LIB_DIR` | `$(PREFIX)/lib` | `libhl.dylib`, `*.hdll` |
| `INSTALL_INCLUDE_DIR` | `$(PREFIX)/include` | `hl.h`, `hl_ffi.h`, `hlc.h`, `hlc_main.c` |

To install under Homebrew's prefix instead:

    make install PREFIX=/opt/homebrew

`hl` is linked with `-rpath @executable_path -rpath $(INSTALL_LIB_DIR)`, so an
installed `hl` finds `libhl.dylib` and the `.hdll` files in
`$(INSTALL_LIB_DIR)` from any working directory. That rpath is baked in when
`hl` is **linked**, not when it is installed, so pass the same `PREFIX` to
both `make` and `make install`. Building with the default prefix and then
installing elsewhere produces an `hl` that cannot start:

    dyld[83852]: Library not loaded: @rpath/libhl.dylib
      Referenced from: <...>/inst/bin/hl
      Reason: tried: '<...>/inst/bin/libhl.dylib' (no such file),
      '/usr/local/lib/libhl.dylib' (no such file), ...

If you have already built, `make clean` or at least `rm hl` before rebuilding
with the new `PREFIX`, since the rpath only changes when `hl` is relinked.

`make uninstall` removes what `make install` copied, and on Darwin
`make install` runs `uninstall` first.

## Running .hl applications

    ./hl program.hl

`hl` resolves a `.hdll` with `dlopen("<name>.hdll")` after trying
`<name>64.hdll` (`src/module.c`). A leaf name like that is found:

- in the current working directory;
- next to the `hl` executable;
- in any directory on `DYLD_LIBRARY_PATH`;
- in `$(INSTALL_LIB_DIR)` after `make install`.

When none of those has it:

    src/module.c(539) : FATAL ERROR : Failed to load library fmt.hdll

`HL_DISABLED_LIBS` takes a comma separated list of library names to refuse to
load; calling a primitive from a disabled library then raises rather than
loading the library.

Useful flags (`hl --help` prints only the usage line; the full list is the
argument parsing in `src/main.c`):

    hl --version
    hl --profile <samples-per-second> program.hl    # writes hlprofile.dump
    hl --debug <port> [--debug-wait] program.hl

## Verifying that you are running native arm64

    file hl
    # hl: Mach-O 64-bit executable arm64

    lipo -archs hl
    # arm64

`lipo -archs` takes one file at a time; `file` accepts several.

For the libraries and their dependencies:

    lipo -archs libhl.dylib
    otool -L sdl.hdll
    file /opt/homebrew/opt/sdl3/lib/libSDL3.0.dylib

For a process, check that it is not running under Rosetta:

    sysctl -n sysctl.proc_translated     # 0 = native, 1 = translated
    arch                                 # arm64
    uname -m                             # arm64

`sysctl.proc_translated` describes the process that reads it, so run it from
the same shell, or from inside the program, rather than assuming it applies
to another process.

## Native library status

Verified by `other/tests/libs/NativeLibs.hx` and
`other/tests/libs/SdlWindow.hx` on the arm64 `hl`; see
`other/tests/hlc/README.md` for how to run them.

| Library | arm64 | Links | Verified behaviour |
| --- | --- | --- | --- |
| `fmt.hdll` | yes | libpng, libturbojpeg, libvorbisfile | `hl.Format.digest` SHA-1 matches; zlib compress/uncompress roundtrip |
| `sdl.hdll` | yes | libSDL3 | window, OpenGL 3.2 core context, shader compile and link, clear, present, `readPixels` |
| `ssl.hdll` | yes | libmbedtls/libmbedx509/libmbedcrypto 3.6.7, Security, CoreFoundation | system trust store loads, `sys.ssl.Digest` SHA-256 matches, real TLS fetch of `https://example.com` |
| `openal.hdll` | yes | libopenal 1.25.2 | device and context created, `AL 1.1 ALSOFT 1.25.2` reported |
| `uv.hdll` | yes | libuv | default loop obtained, `run(NoWait)` returns |
| `sqlite.hdll` | yes | system `/usr/lib/libsqlite3.dylib` | in-memory database, DDL, inserts, queries (SQLite 3.51.0) |
| `mysql.hdll` | yes | none beyond libhl | loads; a connection to a closed port is refused rather than crashing. Not tested against a server |
| `ui.hdll` | yes | none beyond libhl | loads and its natives are callable. `libs/ui/ui_stub.c` is the macOS backend and every entry point is a no-op, so there is no windowing behind it |
| `heaps.hdll` | yes | libc++ | builds and links. Not exercised at runtime here |

Standard library areas that are implemented natively, all verified working:

- `sys.FileSystem`, `sys.io.File`: create, append, read, `stat`, directory
  listing, `fullPath`, delete.
- `sys.io.Process`: `/bin/echo` output captured, exit codes reported.
- `sys.net.Host`, `sys.net.Socket`: loopback listen, accept, connect, and a
  request/response exchange.
- `sys.thread.Thread`, `Mutex`, `Lock`: four threads doing 20000
  mutex-guarded increments each, and `sendMessage`/`readMessage`.
- `haxe.Timer` on the thread `EventLoop`.
- Callbacks from C into compiled code: `Array.sort` comparators,
  `Reflect.callMethod`, closures.
- GC: 200k allocations followed by a major collection, survivors intact.

`hl --profile` works: it samples and writes `hlprofile.dump`.

## HL/C

`haxe -hl out.c` emits C that Apple clang compiles to arm64 against
`libhl.dylib` and the `.hdll` files. Verified with `other/tests/HelloWorld.hx`,
`other/tests/Threads.hx` and `other/tests/hlc/HlcNatives.hx` (fmt, sqlite and
sdl in one executable). No changes to `src/hlc.h` or `src/hlc_main.c` were
needed, and the generated C compiles without architecture specific warnings.

For a program that uses no `.hdll`, the `Makefile` builds it:

    haxe -hl src/_main.c -main HelloWorld -cp other/tests
    make hlc
    ./hlc

The `hlc` rule links only `libhl.dylib`, so a program that uses `.hdll`
libraries has to be linked by hand against the libraries `hlc.json` lists
under `"libs"`:

    cc -O3 -std=c11 -arch arm64 -I src src/_main.c -o app \
       libhl.dylib fmt.hdll sqlite.hdll sdl.hdll ui.hdll \
       -Wl,-rpath,@executable_path

`-Wl,-rpath,@executable_path` is required: `libhl.dylib` and the `.hdll`
files are built with `-install_name @rpath/<name>`.

`other/tests/hlc/README.md` has the full procedure, including how to clean
the generated C afterwards. Note that `haxe -hl out.c` invokes `haxelib` at
the end of the build and prints `Error: This is the first time you are
running haxelib` on a machine where haxelib was never configured; the C is
still generated correctly.

## Hardened Runtime and entitlements

The JIT maps its code pages with `mmap(..., MAP_JIT, ...)`. On Apple Silicon:

- plain RWX `mmap` without `MAP_JIT` is refused with `EPERM`;
- `MAP_JIT` works for the ad-hoc, linker-signed `hl` that `make` produces, so
  a normal build needs no signing at all;
- each thread starts in execute mode and `pthread_jit_write_protect_np`
  toggles write/execute per thread;
- `mprotect` on `MAP_JIT` pages fails.

`src/gc.c` already handles `MAP_JIT` and the write-protect toggling.

Signing `hl` with the Hardened Runtime (`codesign -o runtime`), which is what
an application bundle needs, changes this. Without entitlements it fails
before reaching the JIT, because the Hardened Runtime also turns on library
validation:

    $ codesign -f -s - -o runtime hl
    $ ./hl hello.hl
    dyld[82283]: Library not loaded: @rpath/libhl.dylib
      Referenced from: .../hl
      Reason: tried: '.../libhl.dylib' (code signature in <...> '.../libhl.dylib'
      not valid for use in process: mapping process and mapped file (non-platform)
      have different Team IDs), ...

Re-signing `libhl.dylib` ad-hoc does not help: an ad-hoc signature has no
Team ID, so library validation can never be satisfied that way. With
`com.apple.security.cs.disable-library-validation` the process starts and
then dies in the JIT, because `MAP_JIT` is still refused:

    $ ./hl hello.hl
    SIGNAL 11[Segmentation fault: 11]

With `com.apple.security.cs.allow-jit` as well, it runs normally.
`other/osx/entitlements.xml` carries both, plus `get-task-allow` for
debugger attach, and the `codesign_jit` target applies them:

    make codesign_jit
    # codesign -f -s - -o runtime --entitlements other/osx/entitlements.xml hl

Signing is deliberately not part of `all`.

`com.apple.security.cs.allow-unsigned-executable-memory` is not needed and is
not requested: it is the weaker, non-`MAP_JIT` escape hatch, and the JIT does
not use plain RWX memory.

One extra entitlement is needed by applications that use `openal.hdll`.
Under the Hardened Runtime `alcCreateContext` returns null, and OpenAL then
has no context:

    FAIL  openal.hdll  device + context: alcCreateContext returned null

Adding `com.apple.security.device.audio-input` to the signature fixes it
(OpenAL Soft enumerates capture devices while creating a context). It is not
in `other/osx/entitlements.xml` because `hl` itself does not need it; add it
to your application's entitlements if it uses audio.

## Regression tests

`other/tests/jit/README.md` describes the JIT regression programs added for
the AArch64 backend (conditional moves, callback and native argument
passing, null access, NaN compares, integer edge cases, switch tables,
exceptions, GC stress). `other/tests/libs/EregClobberRepro.hx` is an
end-to-end witness for the conditional-move defect that used to crash
`new sdl.Window(...)`; it prints `done` on a correct build. The Haxe test
suites (`tests/unit`, `tests/threads`, `tests/sys`) and the hxcoro suites
pass on this backend; how they were run is recorded in
[macos-arm64-investigation.md](macos-arm64-investigation.md).

## Known limitations

- **Native debugger backend.** `src/std/debug.c` compiles the mdbg backend
  only under `#if defined(HL_MAC) && defined(__x86_64__)`, and the `Makefile`
  correspondingly leaves `include/mdbg` out of the build on arm64. On arm64
  there is no ptrace fallback either, so `hl_debug_start` returns false and
  the `hl.Debug` process attach, memory and register primitives do not work.
  `hl --debug <port>` starts and the program runs, but attaching a debugger
  client (for example from VSCode) was not tested and is not expected to work
  while those primitives are unavailable.
- **`ui.hdll` is a stub.** `libs/ui/ui_stub.c` is the macOS backend on every
  architecture; its windows, buttons and dialogs do nothing. This is not
  arm64 specific.
- **`heaps.hdll` is not exercised at runtime** here, only built and linked.
- **The CMake build is not covered.** Only the `Makefile` path was verified.
- **`make ARCH=x86_64` is not usable for the `.hdll` files** on a Homebrew
  arm64 installation, because the dependencies are arm64 only.
- **Hot reload (`hl --hot-reload`)** is not implemented by the new JIT on
  any architecture (`hl_jit_patch_method` asserts); not arm64 specific.
- **Dynamic calls are limited to 9 arguments** (`HL_MAX_ARGS` in
  `src/std/fun.c`): `Reflect.callMethod` and `Dynamic`-typed calls with
  more arguments raise `Too many arguments` on every platform.
- **Float-to-int conversion of out-of-range values** saturates on AArch64
  (`FCVTZS`) where x86_64 produces `0x80000000`; Haxe code must not rely on
  either value.
- **Natives called through closures with more than 8 arguments of one
  register class** receive stack arguments in HashLink's internal layout
  rather than the C layout (a design gap shared with the upstream backend
  design; direct calls to natives are correct).
