# Patches carried by `qenesis`

Every commit on `qenesis` that is not in `master` (upstream HashLink), oldest
first. Regenerate the list with

    git log --reverse --format='%h %s' master..qenesis

and update this file as part of every merge into `qenesis`. Merge feature
branches with `git merge --no-ff`, not `--squash`, so the SHAs below stay
valid and each upstreamable fix remains a separate commit.

Upstream status: **sent** (link), **could be sent**, or **fork only**.

| Commit | What it changes | Why | Upstream |
| --- | --- | --- | --- |
| `90799d5d` | Native macOS arm64 support, squashed from `fixes/macos-arm64`: AArch64 backend for the new JIT (upstream PR #932, rebased), fixes for 32-bit CSEL truncating live pointers, c2hl/hl2c stack-argument stride, null-field stub hash and profiler signature, `mbedtls@3` keg in the `Makefile` and `Brewfile`, `codesign_jit` target, CI workflow and `.gitignore` updates, `docs/macos-arm64*.md`, `TASK.md`. The regression tests of `feature/macos-arm64` (`other/tests/jit`, `other/tests/libs`, `other/tests/hlc`) were not included, although the docs refer to them. Also, unintentionally, reverted upstream `ced33c9e` and `04a207d7` in `libs/directx` (restored by `42785dbe`). | Qenesis runs natively on Apple Silicon. | The backend is upstream PR #932 (under review); the fixes on top could be sent to that PR. |
| `fda9a9b3` | `Makefile`: `install` runs `uninstall` first only on Darwin, with the reason documented. | `$(UNAME)==Darwin && ...` was parsed by the shell as an assignment and ran `uninstall` everywhere. | could be sent |
| `228ba8f1` | `Makefile`: `libhl.so` and every `.hdll` linked with `-soname <file name>` on Linux. | Without a soname, an HL/C program linking a `.hdll` by path recorded the absolute path in `DT_NEEDED`. | could be sent |
| `13e2c30a` | CMake: `HL_PREBUILT_DEPS_DIR` for the prebuilt Windows SDL3 / OpenAL / ffmpeg packages; explicit `SDL3_DIR` and `OPENAL_INCLUDE_DIR` win; `SDL3.dll` installed next to `hl.exe`. | The Qenesis bootstrap keeps the source tree read-only, so the packages live outside `include/`. | could be sent |
| `5c4136b8` | `Makefile`: `clean` removes the objects of every JIT backend. | Objects from a build for another `ARCH` survived `make clean`. | could be sent |
| `35ab3176` | `Makefile`: rpath `@executable_path/../lib` / `$ORIGIN/../lib` for `hl` and the `.hdll` files. | The installed tree must stay usable when moved; `hl` could only find `libhl` and the `.hdll` files at the absolute `PREFIX` it was linked for. | could be sent |
| `24133b4c` | CMake: `mysql.hdll` target (`WITH_MYSQL`, off by default on Windows). | CMake installs lacked a library the `Makefile` builds. | could be sent |
| `cfeee051` | CMake on macOS: adds keg-only Homebrew formulae to `CMAKE_PREFIX_PATH`, prefers openal-soft over `OpenAL.framework`. | CMake did not configure out of the box on macOS and linked a different OpenAL than the `Makefile`. | could be sent |
| `42785dbe` | Restores `libs/directx` from `master`. | Undoes the accidental revert of upstream `ced33c9e` and `04a207d7` in `90799d5d`. | fork only (nothing to send; removes a divergence) |
| `83df78a7` | `docs/linux.md`, `docs/windows.md`, updates to `docs/macos-arm64.md`. | Document the vendored-toolchain build on all three platforms. | fork only |
| *this commit* | `PATCHES.md`. | This list. | fork only |
