# ravynOS arm64 build spike (Ellos)

Host: Apple Silicon Mac (arm64), macOS + Xcode 26.6, bmake, 10 cores.
Branch: `arm64-build`. Upstream has never built for arm64 (releases are
x86_64-only) and `ARCH_CONFIGS` was hardcoded `X86_64`.

## Result so far

The build system itself is arch-clean: with one Makefile change it configures
LLVM with `LLVM_TARGETS_TO_BUILD=AArch64` and `aarch64-apple-darwin` triples.
Reached the LLVM compile stage (the long pole) before the current blocker.

## Fixed on this branch

| # | Blocker | Fix |
|---|---------|-----|
| 1 | `ARCH_CONFIGS` hardcoded X86_64 | derive from `MACHINE` (ARM64 on arm64 hosts) |
| 2 | `chown root:wheel` fails (build assumes root) | build with `NO_ROOT=1` (already supported by the Makefiles) |
| 3 | **`TargetConditionals.h` has no `__aarch64__` case** → every arm64 compile hits `#error unrecognized GNU C compiler` | added an arm64 branch (`TARGET_CPU_ARM64`) |
| 4 | `xar`: `openssl/evp.h` not found (not in the macOS SDK) | pick up Homebrew `openssl@3` on Darwin hosts |
| 5 | `dtrace_ctf`: `qsort_r` conflicting types | don't redefine it on macOS (different arg order) |
| 6 | LLVM cmake: `Unknown argument -fno-common` | quote `CMAKE_C_FLAGS` as `CMAKE_CXX_FLAGS` already was |

## Current blocker (architectural, not a one-liner)

LLVM's `Support/Unix/Process.inc` fails: `task_get_exception_ports` /
`task_set_exception_ports` undeclared. Cause: the toolchain force-includes
ravynOS's **target** headers (`Developer/Default.xctoolchain/include`) into
**host** compiles, shadowing the macOS SDK's `mach/*`. Their `mach/task.h` is
MIG-generated at build time, so during the stage-1 host LLVM build it does not
exist yet — a host/target header-separation problem (chicken-and-egg with MIG),
which needs an upstream-shaped fix, not a patch here.

Note `mach/arm/task.h` exists upstream, so some arm groundwork is present.

## Verdict

arm64 is not blocked by anything fundamental *so far*, but it is a long tail of
host-portability work before any bootable artifact exists. Nothing here has been
run; this is a build-system spike only.
