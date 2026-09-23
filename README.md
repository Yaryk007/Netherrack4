# Netherrack4

A PS4 emulator for iOS, built on Linux with [xtool](https://github.com/xtool-org/xtool).

## Build & run

```sh
scripts/build-fexcore-ios.sh [--verify]   # once: build FEXCore for iOS -> ThirdParty/FEXCore/lib/libnrfex.a
xtool dev          # build, sign with your Apple account and install on the connected iPhone
xtool dev build    # build only -> xtool/Netherrack4.app
NR_HOST_TESTS=1 swift test   # run the core test-suite on the Linux host
```

## Layout

| Path | What it is |
| --- | --- |
| `Sources/Netherrack4` | SwiftUI app: library, firmware manager, settings, console, emulation screen (Metal display + touch DualShock 4 + MFi/DualShock/DualSense support) |
| `Sources/NRKit` | Swift wrappers over the core's C API |
| `Sources/NRCore` | The emulator cores (C++17, C API in `include/NRCore.h`) |
| `Sources/NRCore/formats` | PARAM.SFO, PKG (header + metadata entries), PUP (PS4 PUP and SLB2 headers) |
| `Sources/NRCore/loader` | ELF / fake-SELF loader, SCE dynamic tables, NID imports, relocations |
| `Sources/NRCore/cpu` | CPU backend interface, x86-64 interpreter, FEX-Emu backend slot |
| `Sources/NRCore/memory` | Software-translated guest address space |
| `Sources/NRCore/hle` | HLE modules: libc, libkernel, VideoOut, Pad, UserService, SystemService, Sysmodule |
| `scripts/selftests` | x86-64 test programs; expected results are recorded on real hardware |
| `scripts/demo` | Source of the built-in PS4-format demo ELF |

Requires iOS 17.4+ (FEXCore uses `os_sync_wait_on_address`; StikDebug needs 17.4+ too).

## JIT (FEX + StikDebug)

The FEX backend uses the arm64-apple-darwin/iOS port of FEXCore from
[AetherPS4](https://github.com/Leviidev/AetherPS4) (`runtime/sources/fexcore-darwin`, MIT, based on
upstream FEX `f2b679f6`). There is no prebuilt FEXCore in that repo; `scripts/build-fexcore-ios.sh`
fetches the pinned commit, applies `ThirdParty/FEXBridge/patches/0001-embedder-jit-hook.patch`, and
builds it together with `ThirdParty/FEXBridge/fex_bridge.cpp`. `--verify` diffs the fork against upstream
FEX and scans the changes. AetherPS4's prebuilt `BreakpointJIT.framework` is not used: its three
`brk #0xf00d` stubs are reimplemented in `Sources/NRCore/host.cpp`.

JIT needs [StikDebug](https://stikdebug.xyz). The Library's status panel and Settings show whether JIT
is enabled and offer **Enable JIT with StikDebug** (`stikdebug://enable-jit?bundle-id=…&pid=…`, with
`script-name=universal.js` on TXM devices, i.e. iOS 26+ on A15/M2 and newer such as the iPhone 16e).
Executable memory then comes from StikDebug's BRK protocol on TXM devices, or `mprotect(RX)` on older
iOS. The test pattern always uses the JIT when it is enabled; games follow the CPU setting.

## Memory

With **Use all memory iOS allows** (default) the guest gets `os_proc_available_memory()` minus
640 MB of headroom for Metal, the JIT and the UI, capped at 3 GB unless the app was signed with
`com.apple.developer.kernel.increased-memory-limit` (then everything iOS grants, e.g. ~6 GB on an
8 GB device). `Resources/Netherrack4.entitlements` requests that entitlement and extended virtual
addressing. If your signing account refuses them, remove `entitlementsPath` from `xtool.yml` (the app
detects at runtime which ones were granted). The budget is split into a libc heap (¼, max 1 GB) and
PS4 direct memory, and `sceKernelGetDirectMemorySize` reports the real budget to games.

## Status

Working and tested (`NR_HOST_TESTS=1 swift test`, 16 tests):

- x86-64 interpreter: integer ISA, flags, string ops, atomics, common SSE/SSE2, scalar and packed float.
  Six self-test programs match results from a real x86-64 CPU exactly.
- ELF/fake-SELF loading, SCE dynamic tables, NID resolution (`printf` → `hcuQgD53UxM`), JUMP_SLOT/GLOB_DAT/RELATIVE relocations.
- HLE trampolines: backend-independent host calls, including SysV varargs (`printf`).
- VideoOut linear framebuffers → Metal; Pad input from touch controls and hardware controllers.
- Built-in demo ELF runs end-to-end: load → link → print → draw → react to pad input → clean exit.
- PUP/PKG/SFO parsing; encrypted SELFs are detected and refused with a clear message.
- FEX backend: FEXCore links into the iOS app; HLE thunks are `hlt; ret` slots shared by both
  backends; identity-mapped guest memory (tested on the interpreter on Linux).
- Touchpad: expandable swipe surface sending two-finger touch data to `scePadReadState`.

Built but not yet run on a device: the FEX JIT path itself (FEXCore can only run on arm64
Apple hardware). Settings → **Run FEX JIT smoke test** runs FEX's embedding checks on the iPhone.

Not done yet (in rough order):

1. On-device validation of the FEX backend; SIGSEGV/SIGBUS handling (unaligned atomics, SMC tracking).
2. Threads (`scePthread*`), event queues, file I/O (`sceKernelOpen/Read`) and the FreeBSD syscall layer.
3. The GPU: Gnm command buffers and GCN shader → Metal translation, plus detiling of tiled display buffers.
4. LLE loading of decrypted system modules (`.sprx`) from the Firmware tab.
5. Audio (`sceAudioOut`), save data, trophies.
6. PS4 home menu from the imported PS4UPDATE.PUP.

## Legal

Netherrack4 contains no Sony code, keys or firmware. Use only games and system software dumped from a console you own.
