# pybridge

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![Platform: Windows](https://img.shields.io/badge/Platform-Windows%20x86%2Fx64-0078D6.svg)](#) [![x64dbg plugin](https://img.shields.io/badge/x64dbg-plugin-444.svg)](https://github.com/x64dbg/x64dbg) [![CI](https://github.com/EGKrb/pybridge/actions/workflows/ci.yml/badge.svg)](https://github.com/EGKrb/pybridge/actions/workflows/ci.yml)

**pybridge** lets you **drive [x64dbg](https://github.com/x64dbg/x64dbg) from Python** — the GUI or `headless.exe` (x64dbg with no window): breakpoints, single-stepping, registers, memory, API hooks, traces. It is an x64dbg plugin that opens a local TCP server, plus a **standard-library-only Python client**, example scripts and ready-to-use **templates for `headless.exe`**.

🌐 **Website:** <https://egkrb.github.io/pybridge/> · 🇫🇷 [Lire en français](README.fr.md) · [⬇ Latest release](https://github.com/EGKrb/pybridge/releases/latest)

```python
from x64dbg_bridge import Debugger

with Debugger.launch(r"C:\Tools\x64dbg\release") as dbg:      # starts headless.exe
    dbg.init(r"C:\Windows\System32\whoami.exe")

    # Log every DLL load without stopping the program
    dbg.set_breakpoint("kernelbase.LoadLibraryExW", break_=False,
                       log="LoadLibrary({utf16@arg.get(0)})")

    # Break on the first file open, then read the arguments
    dbg.set_breakpoint("kernelbase.CreateFileW")
    if dbg.run(wait=True, timeout=10):
        print("opening:", dbg.read_string(dbg.eval("arg.get(0)"), wide=True))
        print(dbg.regs())
```

| Folder | Content |
|---|---|
| [`src/`](src) | the x64dbg plugin (`pybridge.dp64` / `pybridge.dp32`): a TCP server on **127.0.0.1** that turns JSON requests into x64dbg API calls |
| [`python/x64dbg_bridge.py`](python/x64dbg_bridge.py) | the Python client (standard library only, nothing to install) |
| [`examples/`](examples) | example scripts: API monitor, trace export, unpacking, memory diff, XOR decryption, dump emulation... |
| [`headless/`](headless) | PowerShell launcher and x64dbg script templates for `headless.exe` |

> **Security:** the server only accepts local connections, but any local program that connects to it fully controls the debugger (read/write the debuggee's memory, run commands). Start it only when you need it.

---

## Install

1. Download `pybridge_<version>.zip` from the [latest release](https://github.com/EGKrb/pybridge/releases/latest).
2. Extract it **into the `release` folder of x64dbg** (the one that holds `x64\` and `x32\`). It adds:
   ```
   x64\plugins\pybridge.dp64
   x32\plugins\pybridge.dp32
   pybridge\python\  examples\  headless\
   ```
3. In Python, add `<x64dbg>\release\pybridge\python` to `sys.path` and `from x64dbg_bridge import Debugger`.

The server starts:

- automatically when the `X64DBG_PYBRIDGE_PORT` environment variable is set (this is what `Debugger.launch()` does);
- otherwise with the `pybridge.start [port]` command in x64dbg (decimal port, 27041 by default). Also: `pybridge.stop`, `pybridge.status`, `pybridge.emu` (see [Emulating a dump](#emulating-a-dump-dmp)).

To connect to an x64dbg that is already open (after `pybridge.start`): `Debugger()`.

## Python API

Addresses and values accept an integer or an **x64dbg expression**: `"kernelbase.CreateFileW"`, `"rsp+8"`, `"[rsp]"`, `"mod.base(target.exe)"`. In expressions, numbers are **hexadecimal** by default (`.100` = 100 decimal).

| Category | Methods |
|---|---|
| Session | `launch(x64dbg_dir, arch, gui)`, `init(path, arguments, cwd)`, `attach(pid)`, `stop()`, `close()` |
| Execution | `run(wait, timeout, pass_exceptions)`, `pause()`, `step_into()`, `step_over()`, `step_out()`, `wait(timeout)`, `state()` |
| Registers | `regs()`, `reg(name)`, `set_reg(name, value)` |
| Memory | `read(addr, size)`, `write(addr, data)`, `read_ptr(addr)`, `read_string(addr, wide)`, `is_valid(addr)`, `disasm(addr, count)` |
| Breakpoints | `set_breakpoint(addr, condition, log, command, break_, singleshot)`, `delete_breakpoint(addr)`, `set_hardware_breakpoint(addr, kind, size)` |
| API hooks | `hook_api(api, log, condition, command, break_)` (logs without pausing by default), `dump_memory(addr, size, path)` |
| Anti-anti-debug | `hide_debugger()` (x64dbg's `hide` command: patches the debuggee's PEB), `scyllahide_profiles(ini)`, `scyllahide_set_profile(ini, name)` |
| Traces | `trace(record_file, condition, max_steps, step_over)`, `export_trace(trace, output, fmt, slide)` |
| Commands | `cmd("raw text")`, `command(name, *args)` (automatic escaping), `eval(expr)`, `result` (`$result`) |

pybridge works with **official x64dbg**. One exception: `export_trace()` uses the `TraceExport` command (trace → JSON, CSV, Tenet or Frinet), which exists only in [Fathom](https://github.com/EGKrb/Fathom), the x64dbg-based toolkit pybridge ships with.

## Examples

In [`examples/`](examples) (`--x64dbg <release folder>` or the `X64DBG_DIR` variable picks the x64dbg to use):

| Script | Purpose |
|---|---|
| `api_monitor.py target.exe` | logs API calls (CreateFileW, LoadLibraryExW, VirtualProtect...) without stopping the program |
| `trace_to_json.py target.exe --steps 20000 --out dir` | records a trace from the entry point, exports it to JSON/CSV and prints statistics (needs Fathom) |
| `dump_memory.py --target target.exe --addr "mod.base(target.exe)" --size "mod.size(target.exe)" --out m.bin` | saves a memory range (or `--pid` to attach) |
| `step_log.py target.exe --count 50` | single-steps and prints the registers changed by each instruction |
| `antidebug.py target.exe [--scyllahide-ini ...]` | hides the debugger (PEB) and logs anti-debug probes without blocking them |
| `pe_entropy.py target.exe [--threshold 7.5]` | entropy of the PE sections of the running process, dumps those above the threshold |
| `memory_diff.py target.exe --addr ... --size ... --delay 10` | before/after snapshot of a memory range, lists the modified regions (packers, runtime decompressors) |
| `unpack_dump.py target.exe --out-dir dumps` | dumps every region that `VirtualProtect` flips to PAGE_EXECUTE* (timestamped names + address + size + flag) |
| `xor_decrypt.py target.exe --addr ... --size ... --key ... [--brute]` | reads an encrypted range from the process and XOR-decrypts it (known key or 1-byte brute force) |
| `emulate_dump.py crash.dmp --count 200` | **emulates** a memory dump (stepping, JSON/CSV trace, pybridge server) — see below |

## Emulating a dump (.dmp)

A `.dmp` is a **frozen snapshot** of memory: it does not run, there is no CPU or kernel behind it. `examples/emulate_dump.py` loads the captured memory and the thread registers into the **Unicorn** emulator and runs the code from there. Minidump parsing is standard library only; emulation needs `pip install unicorn` (and optionally `capstone` for disassembly).

```powershell
py emulate_dump.py crash.dmp --list-threads            # threads and their rip
py emulate_dump.py crash.dmp --thread 0 --count 200    # emulated stepping
py emulate_dump.py crash.dmp --from "rsp+8"            # start elsewhere (expression on registers)
py emulate_dump.py crash.dmp --count 5000 --keep-going --trace t.json --trace-csv t.csv
py emulate_dump.py crash.dmp --serve --port 27041      # drive it with x64dbg_bridge.Debugger
```

- **Faithful mode (default):** stops at the **edge of the snapshot** — the first page read that the dump did not capture, or the first system call. On a classic minidump (stack + a few modules) this comes after a few instructions; on a *full dump* it goes much further.
- **`--keep-going`:** removes that stop on a best-effort basis — missing pages mapped as zeros, `syscall`/`int`/`sysenter` skipped, instructions Unicorn cannot decode skipped. **From the first missing byte, the trace is speculative**, not the truth of the process. Reported in the output and by `"skipped": true` in the trace.
- **`--serve`:** exposes the emulator over **the plugin's protocol**, so `x64dbg_bridge.Debugger(port=...)` drives the dump like a live target (`step_into`, `regs`, `read`, `disasm`...), within the limits of the snapshot.

From x64dbg itself, `pybridge.emu "crash.dmp"[, port[, thread[, keepgoing]]]` starts `--serve` in a console. The script is found in `<release>\pybridge\examples\` (or set `X64DBG_EMU_SCRIPT`).

## Templates for headless.exe

```powershell
.\headless\run-headless.ps1 -Template api-log -Target C:\Windows\System32\whoami.exe -X64dbgDir C:\Tools\x64dbg\release
.\headless\run-headless.ps1 -Template dump-module -Target .\program.exe -X64dbgDir C:\Tools\x64dbg\release -Arch x32
```

The launcher replaces `%TARGET%`, `%MODULE%`, `%OUT%` and `%BITS%` in the template, runs `headless.exe -cf` with an isolated configuration folder (the template's `settingset` commands do not change your x64dbg settings) and writes everything to `out\<template>-<date>\` (`script.txt`, `headless.log`, produced files). A template ends with `log "[template] done"`.

| Template | Purpose |
|---|---|
| `api-log` | logs API calls with their arguments, without pausing |
| `trace-export` | traces 50,000 instructions from the entry point and exports them to JSON/CSV/Tenet (needs Fathom's `TraceExport`) |
| `dump-module` | saves the main module (`.mem`) and a full minidump at the entry point |
| `break-on-file` | conditional breakpoint (path containing `.txt`), then automatic actions: log, stack save, minidump |
| `antidebug` | starts the target with `hide` (patched PEB: `BeingDebugged`, `NtGlobalFlag`, heap flags) and logs anti-debug probes |
| `dump-at-delay` | lets the target run, then dumps the main module + a minidump (packers that unpack at startup) |
| `hook-unpack` | logs, without pausing, the typical runtime-unpacking APIs (`VirtualAlloc`, `VirtualProtect`, `WriteProcessMemory`, `NtMapViewOfSection`, `CryptDecrypt`, `BCryptDecrypt`) |
| `unpack-dump` | logs the `VirtualProtect`/`VirtualProtectEx` calls that flip to PAGE_EXECUTE* (mask `0xF0`) and dumps the **last** region (`vprotect-last.bin`); for one dump per call: `examples/unpack_dump.py` |
| `find-pattern` | `findallmem` over the whole memory map: hex pattern with wildcards, or ASCII literal |
| `run-trace` | records 100,000 instructions from the entry point and exports them to JSON and CSV (needs Fathom's `TraceExport`) |
| `hook-apis-extended` | wider version of `api-log` (process/thread control, loader, memory, I/O, registry, crypto) |
| `emulate-dump` | `-Target` is a `.dmp`: starts `pybridge.emu` and serves the emulation on 127.0.0.1:27041 |

The templates read arguments with `arg.get(n)`: they work on x64 and x32 (`-Arch x32`).

## Anti-anti-debug

Two levels, which can be combined:

- **`dbg.hide_debugger()`** (or `hide` in a script) — x64dbg's built-in command. Patches the debuggee's PEB as soon as you have control: `BeingDebugged = 0`, debug bits (`0x70`) of `NtGlobalFlag` cleared, process heap flags normalised. Covers x64 and WoW64. Idempotent; call it after `init()` / `attach()` and before the first `run()`. Enough for basic probes (`IsDebuggerPresent`, `CheckRemoteDebuggerPresent`, direct PEB reads).
- **ScyllaHide** — third-party plugin for advanced probes (`NtQueryInformationProcess`, `NtSetInformationThread`, timing, exceptions...). Put its files (from the ScyllaHide release) in `<x64dbg>\x64\plugins\` and `<x64dbg>\x32\plugins\`, then pick a profile **before** `init()` / `attach()`:

  ```python
  ini = r"C:\Tools\x64dbg\release\x64\plugins\scylla_hide.ini"
  print(Debugger.scyllahide_profiles(ini))          # ['VMProtect x86/x64', 'Themida x86/x64', ...]
  Debugger.scyllahide_set_profile(ini, "Basic")
  ```

These features are meant for analysing binaries you are allowed to study (your own programs, malware samples in a sandbox, CTF challenges...). They only affect the process debugged by that x64dbg instance.

## Protocol (for other languages)

One JSON request per line, one JSON response per line:

```
→ {"id": 1, "method": "mem_read", "params": {"addr": "rsp", "size": 16}}
← {"id": 1, "result": "00000000000000009512A857F77F0000"}
← {"id": 2, "error": "cannot read memory"}
```

Methods: `ping`, `cmd`, `eval`, `state`, `wait`, `run`, `pause`, `step_into`, `step_over`, `step_out`, `regs`, `reg_get`, `reg_set`, `mem_read`, `mem_write`, `mem_valid`, `disasm` (see [`src/api.cpp`](src/api.cpp)). Addresses and registers are returned as hex strings (`"0x7FF6..."`).

## Building

Requirements: Visual Studio 2022 (C++ workload), CMake, Ninja, and the x64dbg **plugin SDK** (the `pluginsdk` folder of an [x64dbg release](https://github.com/x64dbg/x64dbg/releases)). By default the SDK of an x64dbg installed with winget is used; otherwise pass `-DX64DBG_SDK=<pluginsdk>`.

```bat
:: 64-bit, after vcvars64.bat
cmake -B build-x64 -G Ninja -DCMAKE_BUILD_TYPE=Release -DX64DBG_SDK=C:\Tools\x64dbg\pluginsdk
cmake --build build-x64

:: 32-bit, after vcvars32.bat
cmake -B build-x32 -G Ninja -DCMAKE_BUILD_TYPE=Release -DX64DBG_SDK=C:\Tools\x64dbg\pluginsdk
cmake --build build-x32
```

## Part of

pybridge is bundled with [Fathom](https://github.com/EGKrb/Fathom) (reverse-engineering toolkit built on x64dbg), which adds `TraceExport` and the minidump viewer. This repository is its standalone home. To inspect kernel memory next to a pybridge session, see [wineye](https://github.com/EGKrb/wineye).

## License

GPL-3.0-or-later — see [LICENSE](LICENSE). pybridge is a plugin for [x64dbg](https://github.com/x64dbg/x64dbg) (GPL-3.0).
