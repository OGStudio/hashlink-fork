# Native HashLink on macOS ARM64: investigation report

This document records the archaeology, evidence and decisions behind the
`feature/macos-arm64` branch. The user-facing build/run guide is in
[macos-arm64.md](macos-arm64.md).

Starting point: `master` = upstream `HaxeFoundation/hashlink` master at
`b1632a28b6baa05ab8d628b66f93632c2a7f500d` (2026-09-17, "use stack pointer
walk for more accurate profiling"). The fork's `master` and upstream `master`
were identical at the start of the work (0 commits ahead/behind).

Host used for all measurements: Apple Silicon Mac, macOS 26.6.2 (Darwin
25.6.0), Xcode Command Line Tools with Apple clang 21.0.0
(`arm64-apple-darwin25.6.0`), macOS SDK 26.5, Homebrew (arm64) at
`/opt/homebrew`, Haxe 4.3.7 (Homebrew) and Haxe 5.0.0-preview.1 nightly
`4de31b9` (universal binary, used natively as arm64). `sysctl
sysctl.proc_translated` reports `0` for the shells and VMs used, i.e. nothing
below ran under Rosetta unless explicitly stated.

## 1. Baseline (Phase 1)

Built `master` with the Makefile (`make -j8`, then `make -k -j8`), on the
commit above.

| Artifact | Result | Architecture (`file` / `lipo -archs`) |
|---|---|---|
| `libhl.dylib` | builds | arm64 |
| `fmt.hdll`, `heaps.hdll`, `mysql.hdll`, `openal.hdll`, `sdl.hdll`, `sqlite.hdll`, `ui.hdll`, `uv.hdll` | build | arm64 |
| `ssl.hdll` | **fails**: `libs/ssl/ssl.c:25:10: fatal error: 'mbedtls/error.h' file not found` | n/a |
| `hl` | **deliberately skipped** by the build system (see below) | n/a |

Build-system conditions responsible:

- `Makefile` (master): `ARCH ?= $(shell uname -m)`; the `all` target adds
  `$(HL)` only when `ARCH` is not `arm64`, otherwise it prints
  `HashLink vm is not supported on arm64, skipping` (Makefile lines 246-247).
  `install` and `release_osx` skip `hl` on arm64 as well (lines 254, 370).
  `HL_OBJ` hard-codes the x86_64 backend (`src/jit_x86_64.o`).
- `Makefile` (Darwin section): the native debugger helper
  `include/mdbg/*.o` is only linked into `libhl` when `ARCH` is not `arm64`
  (line 192), so `hl.debug` natives are unavailable on arm64 builds.
- `CMakeLists.txt`: `WITH_VM_DEFAULT` is `OFF` when
  `CMAKE_SYSTEM_PROCESSOR MATCHES "arm|aarch64"` (lines 22-25).
- `.github/workflows/build.yml`: arm64 jobs install a shell stub printing
  `Jit is not supported on arm64` instead of `hl` and run the Haxe test suite
  with `--skip-hl-jit`.
- `ssl.hdll` fails because Homebrew's `mbedtls@3` formula is keg-only
  (headers under `/opt/homebrew/opt/mbedtls@3/include`, not
  `/opt/homebrew/include`); the Makefile only adds `$(brew --prefix)/include`.

What happens if the gate is bypassed: `make hl` on master **does compile and
link** on arm64 (the x86_64 backend is plain C that merely emits x86 bytes),
producing an arm64 Mach-O `hl` reporting version 2.0.0. Running any `.hl`
file crashes with SIGSEGV (exit 139): `hl_alloc_executable_memory` calls
`mmap(PROT_READ|PROT_WRITE|PROT_EXEC)` without `MAP_JIT`, which macOS on
Apple Silicon refuses with `EPERM`; the unchecked `MAP_FAILED` is then used
as the code buffer. Even with memory, the generated code would be x86_64.

HL/C on master already works natively: `haxe -hl out/main.c -main HelloWorld`
followed by `cc -arch arm64 -std=c11 -I src -I out out/main.c -L. -lhl`
yields an arm64 executable that prints `Hello world!`.

## 2. Existing AArch64 work (Phase 2)

Search of upstream issues and pull requests (GitHub search
`repo:HaxeFoundation/hashlink aarch64 OR arm64 OR "apple silicon"`),
plus the upstream branches (`hl_interp` is a 2018 interpreter prototype,
irrelevant). Four plausible JIT implementations were found and fetched
locally as remotes `bmdhacks`, `pign` and `boxcat`:

| Candidate | Head | Base (date) | Behind master | JIT generation |
|---|---|---|---|---|
| PR #932 `bmdhacks/hashlink:aarch64-ir` "AArch64 backend for the IR-based JIT" | `54d6959e` (2026-09-11) | `00050cb0` (2026-09-11) | 6 commits | **new IR JIT** (`jit.c` + `jit_emit.c` + `jit_regs.c` + backend) |
| PR #956 `Pign/hashlink:up/arm64-jit` "Add AArch64 JIT backend" | `dd2ccfa2` (2026-07-14) | `64899c10` (2026-07-13) | 192 commits | old monolithic `jit.c` (parallel `jit_arm64.c`) |
| PR #895 `bmdhacks/hashlink:aarch64-jit` "AArch64 JIT backend" | `5d034d79` (2026-03-06) | `19c4c225` (2026-03-06) | 249 commits | old monolithic `jit.c` (split into `jit_x86.c`/`jit_common.c` + `jit_aarch64*.c`) |
| PR #857 `studio-boxcat/hashlink:bmd-aarch64` "fix arm64 jit" | `5cd677b1` (2025-12-21) | `b0744ba5` (2025-12-16) | 291 commits | same lineage as #895 plus an LLVM AOT backend |

The decisive context: upstream merged a complete JIT rewrite ("Jit2", PR
#911/#968, "HL2 pre release", 2026-09-01). The maintainer declined #895 on
2026-03-14 explicitly because of this rewrite ("when we have this done for
x86_64, it will be much easier to add ARM support"). The new pipeline is
`hl_emit_function` (HL opcodes to an SSA-like IR) -> `hl_regs_function`
(architecture-agnostic register allocation driven by `regs_config`) ->
`hl_codegen_*` (backend). Only #932 targets this pipeline; the other three
implement the pre-rewrite 8-function JIT interface and cannot be applied to
current master without a rewrite of their own.

### 2.1 PR #932 (selected)

- Files: `src/jit_aarch64.c` (2055 lines), `src/jit_aarch64_emit.c` (847),
  `src/jit_aarch64_emit.h` (242), plus small changes to `jit.h`, `jit.c`,
  `jit_regs.c`, `gc.c`, `module.c`, `profile.c`, `hl.h`, `Makefile`,
  `CMakeLists.txt`, CI. 3 commits, +3383/-52.
- Approach: implements the backend contract used by `jit_x86_64.c`
  (`hl_jit_init_regs`, `hl_codegen_init/function/flush_consts/final`).
  Register classes: X0-X14 scratch (X5 remapped to a logical id because the
  IR reserves id 5 for the stack register), X19-X28 persist, V0-V7 and
  V16-V28 scratch, V8-V15 persist, X15-X17 and V29-V31 backend temporaries.
  Frame: `STP X29,X30` + `MOV X29,SP`; every IR `PUSH` moves SP by 16 so SP
  stays 16-byte aligned (`cfg->stack_arg_size = 16`); HL->native stack args
  follow the platform ABI (`NATIVE_STACK_LAYOUT_APPLE_ARM64` packs at natural
  size on macOS, 8-byte slots elsewhere). Constants and jump tables live in
  a per-module pool addressed with ADRP+LDR patched in
  `hl_codegen_flush_consts`; HL->HL calls are direct `BL`; CALL_PTR uses
  a pool-loaded address + `BLR`. c2hl/hl2c trampolines mirror the x86
  ones. `module_capture_stack` walks the X29 chain on AArch64 instead of
  the heuristic stack scan (which produces false positives from `STP
  X19,X20` spills).
- macOS: `MAP_JIT` in `hl_alloc_executable_memory`, per-thread
  `pthread_jit_write_protect_np` toggling (write mode at allocation, execute
  mode in the new `hl_flush_executable_memory` which also calls
  `__builtin___clear_cache`), added at the maintainer's request instead of
  platform `#ifdef`s in `jit.c`.
- Coverage/tests: the author reports the full Haxe unit suite passing and
  Dead Cells running on Asahi Linux (M2); CI is enabled for the arm64 JIT.
  The maintainer was actively reviewing on 2026-09-20 (naming of the new
  `regs_config` fields, keeping `jit_regs.c` architecture-neutral, a
  `hl_resolve_module` helper for `module.c`), i.e. the PR is on the path to
  being merged, which matters for keeping this fork mergeable.
- Rebase onto `b1632a28`: clean except `src/profile.c`, where upstream
  changed `get_thread_stackptr()` to return `pc`/`fp` out-parameters after
  the PR was rebased; fixed in commit `d01c6701`.
- Verified here (native arm64 build, before any fixes): hello world,
  `other/tests/Threads.hx` (13/13 runs), the hxcoro suites (304/304 tests,
  1226 assertions; 15/15 call-stack tests). Defects found by our audit are
  listed in section 4.

### 2.2 PR #956 (Pign) - rejected

Built here (`make libhl.dylib` + `make hl HL_OBJ=...` because its Makefile
was never taught about the backend) and run: hello world, threads, a
closure/exception/`Array.sort` test, a 2M-object GC stress and a
12-Int/10-Float direct call all pass. Rejected because:

- targets the pre-rewrite JIT; porting it to the IR pipeline means
  rewriting it;
- no register allocator: every virtual register lives in a fixed
  `[FP - 8*(i+1)]` slot and every opcode loads/stores through X9/X10/V16/V17
  (slow by design; the `Threads.hx` sample stalled 53 s in 1 of 5 runs, cause
  not determined);
- HL->C wrappers are not implemented (`get_wrapper` returns NULL, so
  `hl_make_fun_wrapper` produces NULL closures), C->HL callbacks reject more
  than 8 args per class, `BL` out of range aborts (no veneers),
  `hl_jit_patch_method` is a stub, HF32 arithmetic uses double-precision
  encoders ("approximate" per the author), `OJULt/OJUGte` on floats fall
  through to an unconditional branch, Apple stack-arg packing is not
  implemented, and `hl_jit_free` ignores `can_reset`.

Reused from it: the idea of shipping an entitlements file and a codesign
target for the Hardened Runtime case (see section 3). Its
`hl_jit_thread_init()` (flipping new threads to execute mode) was verified
to be unnecessary on macOS 26 (section 3).

### 2.3 PR #895 (bmdhacks, old JIT) - rejected

Builds with plain `make` on this Mac. Hello world, threads, closures and
GC tests pass, but a direct call of a function with 12 Int and 10 Float
parameters returns `278` instead of `490.5`, deterministically: the branch
applies Apple's byte-packed stack layout to JIT-to-JIT calls (the
`is_native` flag that should gate it is computed and never read) while the
callee assumes 8-byte slots. Also: HF32 callback args marshalled as f64,
`OJSGte` on floats taken for NaN (differs from x86), virtual calls silently
drop args beyond 8, unimplemented opcodes only print a warning. Superseded
by the same author's #932.

### 2.4 PR #857 (studio-boxcat) - rejected

Independent fork of the same early code as #895 with 20 fix commits, a
5k-line LLVM AOT backend, a CRLF->LF rewrite of `gc.c`/`hl.h` (which would
make any upstream merge painful) and author-reported broken stack traces
("All unit tests pass except stack traces"). No Apple stack packing at
all, C->HL trampoline pushes 16-byte slots while callees read 8-byte
slots, X16/X17 in the allocatable pool although every call clobbers X17.
Not built (static analysis only).

### 2.5 Decision

Outcome 2 of the task's list: **rebase PR #932 onto current master and fix
what the audit finds**, keeping every change inside the JIT/platform layer
so the branch stays a superset of the upstream PR. Writing a new backend
was not considered: #932 already implements the current pipeline and is
under upstream review.

## 3. Apple Silicon JIT memory rules (Phase 3)

Measured with a small C program (`mmap`/`pthread_jit_write_protect_np`
probe, run as an ad-hoc signed binary, as a Hardened Runtime binary, and as
a Hardened Runtime binary with the `com.apple.security.cs.allow-jit`
entitlement), on macOS 26.6.2:

| Probe | ad-hoc (linker signed, what `make` produces) | Hardened Runtime, no entitlement | Hardened Runtime + allow-jit |
|---|---|---|---|
| `mmap(RWX)` without `MAP_JIT` | `EPERM` | `EPERM` | `EPERM` |
| `mmap(RWX \| MAP_JIT)` | ok | **`EINVAL`** | ok |
| write with the thread in its default state | faults (default is execute mode) | n/a | faults |
| write after `pthread_jit_write_protect_np(0)` | ok | n/a | ok |
| execute while still in write mode | faults | n/a | faults |
| execute after `pthread_jit_write_protect_np(1)` | ok | n/a | ok |
| write while in execute mode | faults (W^X enforced per thread) | n/a | faults |
| new `pthread` calling JIT code without any toggle | ok (threads start in execute mode) | n/a | ok |
| `mprotect(RW)` / `mprotect(RX)` on a `MAP_JIT` page | `EACCES` | n/a | `EACCES` |

Consequences for the runtime, all satisfied by the branch:

- executable memory must be allocated with `MAP_JIT` (`src/gc.c`);
- the emitting thread toggles to write mode before filling the buffer and
  back to execute mode after the final patch (`hl_alloc_executable_memory`
  / `hl_flush_executable_memory`, called at the end of `hl_jit_code`); no
  JIT code runs on that thread in between, and other threads are unaffected
  because the toggle is per thread;
- the instruction cache must be invalidated for the whole range including
  the constant pool (`__builtin___clear_cache`, which on Apple maps to
  `sys_icache_invalidate`);
- nothing may write to the code region afterwards: the only late writers
  in the tree are `hl_jit_patch_method` (hot reload), which is
  `jit_assert()` in the new JIT on every architecture, and the external
  debugger (`hl.debug` natives / `include/mdbg`), which is not built for
  arm64;
- no thread bookkeeping is needed (threads start in execute mode), so
  Pign's `hl_jit_thread_init()` is not needed;
- the default `make` output (ad-hoc, linker-signed, no Hardened Runtime)
  needs no entitlement. A Hardened Runtime build (required for notarized
  distribution) must be signed with `com.apple.security.cs.allow-jit`,
  otherwise `mmap` fails with `EINVAL` and `hl` cannot start; see
  [macos-arm64.md](macos-arm64.md) for the codesign target.

## 4. Audit findings on the selected backend (Phase 4)

Method: static audit of `src/jit_aarch64*.c` against the IR contract
(`jit_emit.c`, `jit_regs.c`) and the x86_64 reference backend, AAPCS64 and
the Apple ARM64 ABI, plus targeted Haxe programs run both on the native
arm64 build and on an x86_64 build of the same sources executed under
Rosetta purely as an oracle (`arch -x86_64 ./hl`), and the Haxe unit test
suite (nightly Haxe, `tests/unit/compile-hl.hxml`).

Defects found and fixed (each with a regression test under
`other/tests/jit/`, see its README):

| # | Defect | Symptom | Fix |
|---|---|---|---|
| 1 | `emit_cmov_arm` selected `sf` from the value width, so an `M_I32` conditional phi move became `CSEL W20,W22,W20`, zero-extending and truncating the 64-bit pointer that the fall-through path still owned in X20 (the FP path had the same hazard with `FCSEL S`). The x86_64 backend always performs CMOV at `M_PTR` width. | `haxe.CallStack.exceptionStack()` segfaulted, which killed the Haxe unit suite in its first test (`TestCallStack`). | `173deffb`: always select at full register width; source still materialised at its own width. Test `CMov`. |
| 2 | c2hl trampoline copied C->HL stack arguments from `&vargs.stack[15]` downwards with an 8-byte stride (the convention before upstream `58fadd4c`), while `callback_c2hl` writes `stack[0..N-1]` leftmost-first and the callee reads argument k at `[FP+16+k*16]`. | `Reflect.callMethod(null, f9, [1..9])` returned 330702684 instead of 285 (x86_64 reference: 285); 9 Float arguments likewise. | `440f8266`: copy `stack[0..N-1]` upward to `[SP+k*stride]` with `stride = cfg.stack_arg_size`. Test `CallbackArgs`. |
| 3 | `callback_hl2c` (`src/jit.c`) advanced over caller stack arguments by 8 bytes although the AArch64 backend pushes them with a 16-byte stride. | A `Dynamic` closure cast to a typed 9-argument function crashed in `Std.string`. | `a19ec560`: `hl_jit_code` records `cfg.stack_arg_size`; the wrapper uses `max(natural size, stride)` (architecture-neutral, mirrors `get_stack_size` in `jit_regs.c`; unchanged behaviour on x86_64 where the stride is 0). Test `CallbackArgs`. |
| 4 | The null-field stub assumed the field hash was in W0; the IR emits `PUSH_CONST hash; PUSH_ADDR site; CALL_PTR`, and x86 loads the hash from `[RBP+16]`. | `o.x` on a null object reported `Null access .???`. | `5a1e683f`: `LDR W0,[FP,#16+stride]` after the stub prologue. Test `NullAccess`. |
| 5 | Latent issues found by reading, no observed failure: XCHG with one spilled operand aborted codegen; FP loads/stores could pick X16 as offset temporary while X16 was the base; the ADD/SUB-immediate fast path skipped UI8/UI16 truncation; `MOV [slot] <- MK_STACK_OFFS` stored FP instead of FP+offset; only 64 bytes were reserved per IR op although a spilled div/mod expands to ~100 bytes before the overrun check. | none observed | `51e5394f`. |

Areas audited and found correct (evidence in the regression tests and in
the differential runs against the x86_64 reference): argument and return
registers (X0-X7/V0-V7/X0/V0, X18 excluded, X15-X17 backend temporaries);
sub-word zero/sign extension in both directions (`NativeArgs`); frame
layout (`STP X29,X30` / `MOV X29,SP`, 16-byte persist slots, SP 16-byte
aligned at every call); HL->HL calls with more than 8 integer/float
arguments (`ManyArgs`); Apple natural-size packing of HL->native stack
arguments, checked with a clang-built test hdll (`NativeArgs`); the hl2c
register buffer layout; direct `BL` calls with the +/-128 MB range check,
ADRP+ADD/LDR constant-pool and jump-table relocations (`SwitchTable`);
debug trampolines (`hl --debug <port>` runs of the suites); float compares
with NaN, division/modulo special cases (`x/0 = 0`, `INT_MIN/-1 =
INT_MIN`), shift counts, conversions (`NaNCompare`, `IntOps`, `Int64Ops`);
exceptions across JIT, native and callback frames (`Exceptions`); GC roots
(values live across calls are spilled or in callee-saved registers captured
by `setjmp` in `gc_save_context`; `GcStress`); the W^X sequence (no writer
after `hl_flush_executable_memory`).

Not verified here: the Linux/AAPCS64 stack layout branch (no Linux
machine), debugger stepping (the `hl.debug` natives are not built for
macOS arm64), C->HL calls with more than one stack argument (unreachable
through the public API because of `HL_MAX_ARGS`), hot reload (asserts on
every backend of the new JIT).

Known residual differences from x86_64, kept out of scope: natives called
through closures (`CALL_REG`) with more than 8 arguments of one class use
the HL 16-byte convention rather than the C layout (pre-existing design
gap, also present on other platforms); float-to-int conversion of
out-of-range values saturates (FCVTZS) where x86 yields `0x80000000`;
HF32 values through hl2c wrappers travel as double (identical on x86_64).

Upstream review comments on PR #932 (field naming in `regs_config`, keeping
`jit_regs.c` free of architecture comments, `MAX_CALL_ARGS`, a
`hl_resolve_module` helper) were deliberately not applied here so that
the fork's diff against the PR stays limited to genuine fixes; they should
be picked up when the PR lands upstream.

Observed on both architectures (not arm64 defects, reported for
completeness):

- Rethrowing from a `catch` block loses a local assigned inside that catch
  block when the outer `catch` reads it (`nestedCaught` stays `false` in
  `other/tests/jit/...`; the Haxe interpreter prints `true`). Both the
  native arm64 build and the x86_64 reference build show it, so it is an
  upstream IR/register-allocation issue (`CATCH` spills at the catch label,
  after `longjmp` restored the callee-saved registers). Not fixed here.
- `HL_MAX_ARGS` is 9 (`src/std/fun.c`), so `Reflect.callMethod` and
  `Dynamic` calls with 10+ arguments raise `Too many arguments` on every
  platform.

## 5. Validation summary (Phase 5)

All runs below used the native arm64 `hl` built from the branch head with
`make -j8` (`file`: Mach-O arm64, `sysctl.proc_translated` = 0). Test
programs were compiled with Haxe 4.3.7 unless stated; the Haxe repository
suites (`4de31b9`, 2026-09-17) need the Haxe 5 nightly and were compiled
with `-lib utest` replaced by the utest git checkout on the class path plus
`-D utest=1.13.2` (the nightly's `haxelib` binary needs libneko, which is
not installed here).

| Step | Program(s) | Result |
|---|---|---|
| 1-2 build, trivial bytecode | `other/tests/HelloWorld.hx` | ok |
| 3-9 ints, floats, branches, calls, recursion, objects, arrays, allocation | `other/tests/jit/IntOps`, `Int64Ops`, `NaNCompare`, `SwitchTable`, `ManyArgs`, Haxe `tests/unit` | ok (20/20 runs each for the jit tests) |
| 10-11 allocation, GC | `other/tests/jit/GcStress` (20/20), page-allocator stress (400 x 8 MB pages, 3.5 GB allocated, forced majors) | ok, no `GC Page HASH collide` |
| 12-13 closures, virtual/dynamic calls | `other/tests/jit/CMov`, `CallbackArgs`, Haxe `tests/unit` | ok |
| 14 exceptions | `other/tests/jit/Exceptions` (20/20), `NullAccess`, `haxe.CallStack.exceptionStack()` repro | ok |
| 15-16 native calls, C->HL callbacks | `other/tests/jit/NativeArgs` (with a clang-built test hdll), `CallbackArgs`, `other/tests/libs/NativeLibs.hx` | ok |
| 17 threads | `other/tests/Threads.hx` (20/20 runs, all under 1 s), Haxe `tests/threads` (135 assertions) | ok |
| 18-20 stress GC, GC + exceptions, GC + natives | `GcStress`, `Exceptions`, `NativeLibs` GC section, `other/tests/jit/stress/` (see below) | ok |
| 21 existing suites | Haxe `tests/unit` with and without `-D analyzer-optimize`: 11830 assertions, all ok, 10 consecutive runs; `tests/threads`: 135 ok; `tests/sys`: 967/969 (the two `testCommand` failures shell out to `haxelib` and fail identically on the x86_64 reference); hxcoro `tests`: 304/304 (1226 assertions); hxcoro `callstack-tests`: 15/15; `tests/misc/cross/eventLoop`: identical output to x86_64; the same suites under `hl --debug <port>` (debug code generation) | ok |

Before the fixes of section 4 the unit suite died in its first test
(`TestCallStack`, SIGSEGV) and `Reflect.callMethod` with 9 arguments
returned garbage; the hxcoro suites already passed.

Profiler: `hl --profile <rate>` samples and writes `hlprofile.dump` on
arm64 (2-3 samples for a 200 ms program at 10 Hz, same count as the
x86_64 reference).

Progressive stress programs (`other/tests/jit/stress/`, 23 programs, each
also run on the x86_64 reference build so that only differences are
attributed to the backend): on the fixed backend every program passes
(GC, thread and exception programs 20/20 runs; `GcStorm` alone allocates
for about 3.5 s per run), with one intentional exception, `FloatArith`,
whose 9 failing checks are the float-to-int saturation difference listed
in section 4. On the pre-fix build (`d01c6701`) the same programs found
exactly the two defects fixed in section 4 (`Exceptions` crashed in
`haxe.CallStack.exceptionStack()`, `Closures` returned garbage for the
9th integer argument of `Dynamic` calls), and nothing else.

## 6. Native libraries and HL/C (Phases 6 and 7)

See [macos-arm64.md](macos-arm64.md) for the per-library status table and
the HL/C procedure.

Summary of what was verified on the native `hl` (details and the exact
programs in `other/tests/libs/` and `other/tests/hlc/`):

- `make -j8` builds `hl`, `libhl.dylib` and all nine hdlls as arm64 with 0
  errors once the Makefile locates the keg-only `mbedtls@3`
  (commit `55125e9e`); every linked Homebrew dylib is arm64.
- Runtime: fmt (digest, zlib), sdl (window, OpenGL 3.2 core context,
  shaders, present, `readPixels`; `new sdl.Window(...)` works once the
  CMOV fix is in), ssl (trust store, digest, a real TLS fetch), openal
  (device and context), uv, sqlite, mysql (loads), ui (loads, but it is
  the stub backend on macOS), filesystem, processes, sockets, threads,
  timers, callbacks, GC. `heaps.hdll` builds and links but was not
  exercised (Heaps is out of scope).
- HL/C: HelloWorld and Threads through `make hlc`, and a fmt+sqlite+sdl
  program linked by hand, all arm64 and working without any change to
  `src/hlc.h` or `src/hlc_main.c`.
- Hardened Runtime: measured that `codesign -o runtime` needs both
  `com.apple.security.cs.allow-jit` and
  `com.apple.security.cs.disable-library-validation`; `other/osx/entitlements.xml`
  carries them and `make codesign_jit` applies them (commit `9354ebb0`).
  OpenAL under the Hardened Runtime additionally needs
  `com.apple.security.device.audio-input`.

## 7. Provenance of reused code

- `src/jit_aarch64.c`, `src/jit_aarch64_emit.c`, `src/jit_aarch64_emit.h`
  and the related changes to `src/jit.h`, `src/jit.c`, `src/jit_regs.c`,
  `src/gc.c`, `src/module.c`, `src/profile.c`, `src/hl.h`, `Makefile`,
  `CMakeLists.txt`, `.github/workflows/build.yml`: Brian Degenhardt
  (bmdhacks), upstream PR #932, commits `873fe032`, `084a6c02`, `54d6959e`
  of `bmdhacks/hashlink:aarch64-ir`, rebased here as `3a864478`,
  `6961d44b`, `03a841ea` with authorship preserved. MIT, same as HashLink.
- Everything after those three commits on `feature/macos-arm64` was written
  for this fork; each commit message states what it fixes.
- No code was taken from #956, #895 or #857.
