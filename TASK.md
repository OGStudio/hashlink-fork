# Task: Native HashLink on macOS ARM64

## Repository

Work in:

`https://github.com/OGStudio/hashlink-fork.git`

This repository is an OGStudio fork of upstream HashLink. Its `master` branch is intended to remain synchronized with current upstream `HaxeFoundation/hashlink`.

Clone/use the repository locally and create a dedicated local feature branch, for example:

`feature/macos-arm64`

Reference upstream as:

`https://github.com/HaxeFoundation/hashlink.git`

Do **not** develop directly on `master`.

## Remote-write policy

You may:

* fetch from remotes;
* inspect upstream branches, commits, issues, and pull requests;
* create local branches;
* create local commits;
* rebase/cherry-pick locally where appropriate.

You must **never**:

* run `git push`;
* force-push;
* create or update remote branches;
* create pull requests;
* merge anything remotely;
* modify GitHub repository settings, issues, releases, tags, or other remote state.

All implementation work must remain local.

Create clean local commits as work progresses. The repository owner will inspect and push the work manually when it is ready.

---

## Goal

Make the normal HashLink development/runtime workflow work natively on Apple Silicon macOS:

```text
Haxe
  ↓
.hl bytecode
  ↓
native arm64 `hl`
  ↓
HashLink runtime + HDLLs
```

The primary target is a native `hl` VM capable of executing `.hl` bytecode without Rosetta.

This task covers:

* HashLink ARM64/AArch64 runtime support;
* JIT support;
* macOS ARM64 integration;
* native HashLink libraries.

It does **not** cover:

* Heaps;
* Metal;
* Vulkan;
* SDL_GPU renderer support;
* Heaps graphics architecture.

---

# Critical constraint

Do **not** begin by implementing a new AArch64 JIT from scratch.

There are already existing/recent AArch64 efforts around HashLink.

Investigate those first.

The first phase is technical archaeology and validation, not implementation.

---

# Phase 1 — Establish the baseline

Build current `OGStudio/hashlink-fork:master` on an Apple Silicon Mac.

Precisely determine what currently works and what does not.

Verify architectures rather than assuming them.

Document at least:

* whether `libhl` builds as arm64;
* which `.hdll` libraries build as arm64;
* whether `hl` itself is deliberately omitted;
* build-system conditions responsible for this behavior;
* whether HL/C-generated programs can already be compiled natively;
* macOS-specific failures encountered.

Use `file`, `lipo`, `otool`, or equivalent tools to verify that binaries really are arm64.

Do not accidentally validate through Rosetta.

Record the exact starting commit of `master`.

---

# Phase 2 — Investigate existing AArch64 work

Search upstream HashLink history, pull requests, forks, and relevant experiments for existing AArch64 JIT implementations.

For each plausible implementation determine:

* base commit / age;
* architectural approach;
* JIT coverage;
* missing opcodes/features;
* ABI implementation;
* floating-point support;
* native-call support;
* callbacks;
* exceptions;
* GC interaction;
* stack walking;
* threading;
* macOS-specific support;
* test coverage;
* known correctness problems;
* divergence from current upstream.

Do not rank candidates by code size or superficial completeness.

Run available tests or construct targeted tests where necessary.

Produce:

`docs/macos-arm64-investigation.md`

The report should conclude with a technically justified implementation strategy.

Possible outcomes include:

1. reuse one implementation mostly unchanged;
2. rebase one onto current master;
3. combine specific pieces from several;
4. implement missing functionality on top of an existing backend;
5. conclude that none is sufficiently usable and document why.

Writing a completely new backend is the last resort.

---

# Phase 3 — macOS ARM64 JIT requirements

Explicitly audit Apple-Silicon-specific JIT behavior.

At minimum investigate and correctly handle where applicable:

* executable memory allocation;
* `MAP_JIT`;
* W^X requirements;
* writable/executable transitions;
* instruction-cache invalidation;
* Apple Hardened Runtime implications;
* JIT-related entitlements if relevant to packaged applications.

Correct generated instructions alone are not sufficient evidence that the JIT is correct on macOS.

---

# Phase 4 — AArch64 correctness

Audit the chosen backend against the actual AArch64/macOS ABI.

Pay particular attention to:

* integer argument registers;
* floating-point/vector argument registers;
* return values;
* stack alignment;
* caller/callee-saved registers;
* stack frames;
* structure/value passing where applicable;
* tail calls;
* indirect calls;
* closures;
* virtual calls;
* varargs/native calls where applicable;
* generated-code relocation.

Also validate interaction with HashLink runtime behavior:

* GC roots;
* stack scanning;
* allocations;
* exceptions;
* callbacks from C into HashLink;
* calls from generated code into native primitives;
* threads;
* synchronization;
* stack traces;
* debugging/profiling paths where applicable.

Avoid architecture-specific hacks outside the proper JIT/platform layer.

---

# Phase 5 — Progressive validation

Do not jump directly from “hello world works” to Heaps.

Validate progressively:

1. native ARM64 `hl` builds;
2. trivial `.hl` bytecode executes;
3. integer arithmetic;
4. floating-point arithmetic;
5. branching;
6. function calls;
7. recursion;
8. object access;
9. arrays;
10. allocations;
11. garbage collection;
12. closures;
13. virtual/dynamic calls;
14. exceptions;
15. native primitive calls;
16. callbacks from C to HashLink;
17. threads;
18. repeated/stress GC;
19. GC combined with exceptions;
20. GC combined with native calls;
21. existing HashLink test suites.

Where a bug is found, add focused regression coverage whenever practical.

For GC/JIT-sensitive behavior, run repeated/stress tests rather than relying on one successful execution.

---

# Phase 6 — Native libraries

Once the VM is sufficiently stable, validate the standard HashLink native libraries relevant to normal development.

Build them as ARM64.

Do not introduce Heaps-specific patches.

Pay particular attention to modules exercising:

* dynamic loading;
* SDL;
* OpenGL;
* filesystem APIs;
* networking;
* threads;
* native callbacks.

Document unsupported libraries rather than silently skipping them.

---

# Phase 7 — HL/C fallback

Separately evaluate:

```text
Haxe
  ↓
HashLink C output
  ↓
Apple Clang
  ↓
native arm64 executable
```

Determine:

* whether it works;
* which HashLink libraries work;
* which fail;
* whether generated applications are genuinely native arm64;
* whether platform patches are required.

HL/C is useful as a deployment fallback.

It is **not** a substitute for a working `.hl -> hl` development loop.

---

# Git discipline

Keep `master` untouched except for explicitly syncing/fetching upstream state when necessary.

Perform implementation on the local feature branch.

Create small coherent local commits, for example:

```text
build: enable arm64 VM target
jit: add AArch64 call lowering
runtime: support macOS JIT memory protection
tests: add arm64 native-call coverage
docs: document macOS arm64 build
```

If upstream patches are reused, preserve attribution and provenance.

Do not:

* commit generated artifacts;
* commit local SDK paths;
* hardcode machine-specific paths unnecessarily;
* add Rosetta as a dependency;
* hide unrelated refactoring inside functional commits.

## Absolute rule

**Do not push any commit or branch to any remote.**

The final state of this task must exist only in the local repository.

---

# Deliverables

Produce locally in `OGStudio/hashlink-fork`:

### Code

A working ARM64 implementation on the dedicated local branch.

### Tests

Regression coverage for newly supported/fixed behavior where practical.

### Documentation

Create:

`docs/macos-arm64.md`

covering:

* prerequisites;
* build commands;
* running `.hl` applications;
* verifying native architecture;
* known limitations.

And:

`docs/macos-arm64-investigation.md`

covering:

* existing AArch64 implementations investigated;
* evidence gathered;
* selected implementation;
* reused code and provenance;
* rejected approaches and rationale.

### Local commit history

All completed work must be committed locally.

The working tree should be clean at handoff unless explicitly documented otherwise.

### Handoff for Heaps

Provide:

* local branch name;
* exact commit SHA considered ready for integration;
* build instructions;
* required dependencies/environment;
* known limitations.

Do not push this branch.

The repository owner will decide whether and when it should be published.

---

# Acceptance criteria

The task succeeds when an Apple Silicon Mac can, without Rosetta:

```text
build arm64 HashLink
        ↓
run native arm64 `hl`
        ↓
execute nontrivial `.hl`
```

and the following have been meaningfully exercised:

* GC;
* exceptions;
* closures;
* native calls;
* native callbacks;
* floating-point code;
* threads.

Relevant HashLink tests should pass.

Remaining failures must be documented precisely.

The implementation should be sufficiently clean that it could plausibly be proposed upstream later.

---

# If full success is not possible

Do not disguise partial support as completion.

Deliver instead:

* the strongest working implementation;
* failing regression tests/reproductions;
* precise blockers;
* suspected ownership of each blocker;
* the most promising existing AArch64 implementation;
* concrete next steps.

Commit the useful partial work locally.

A high-quality blocker analysis is preferable to unsafe JIT code that merely passes simple demos.
