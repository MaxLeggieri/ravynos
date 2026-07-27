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

## Result: toolchain builds; XNU kernel is the wall

**The full Apple-style toolchain builds for arm64** from this tree on an Apple
Silicon macOS host: clang, lld, tapi, the cctools suite, `ld64`, `iig`, and the
CTF tools. ~23 fixes (below), almost all macOS-host portability; only
`TargetConditionals.h` was a genuine arm64 gap.

**Hard wall: the XNU kernel.** `world` then enters `Kernel`, and XNU's arm64
build supports only specific Apple SoCs — `SUPPORTED_ARM64_MACHINE_CONFIGS =
T7000 T7001 S8000 S8001 T8010 T8011 BCM2837` (A8/A9/A10 iPhone chips + Raspberry
Pi 3). There is **no generic-arm64 / QEMU-virt board**, and the build drives
`EMBEDDED_DEVICE_MAP` (Apple's proprietary embedded-device database) to resolve a
board to its arch. So building this kernel targets real Apple/embedded hardware,
not a VM — it cannot produce a QEMU-bootable arm64 kernel from this tree.

**Strategic conclusion (for the Ellos effort that motivated this):** irrelevant.
Ellos already has a working arm64 kernel (FreeBSD/NextBSD). The only part of
ravynOS worth harvesting is its *userland* — the Cocoa/AppKit frameworks and the
X11-free WindowServer — but those sit behind the XNU kernel-header dependency in
`world` (Libraries pull kernel headers from the SDK). Extracting them for arm64
would mean decoupling the framework build from XNU's SDK population — a separate,
larger effort, and a Mach-O-vs-ELF integration question on top.

## Verdict

arm64 has no *toolchain* blocker — it builds. The blocker is the XNU kernel,
which on arm64 is Apple's embedded build system with no generic/VM target. That
is a design wall, not a bug to patch. Nothing was booted; this is a build spike.
