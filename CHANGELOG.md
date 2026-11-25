# Changelog

## [1.1.0] - 2026-11-25

- Client: emulation with [Fathom](https://github.com/EGKrb/Fathom)'s emulator:
  `emulate_call()` (a function of the debuggee run in the emulator, ExtraCmds
  `emu.call`), `emulate_run()` (the thread run on in the emulator, `emu.run`,
  with an optional trace for Time Travel), `emulator()` (a
  `dumpulator.Dumpulator` of the debuggee's current state in the Python
  process) and `minidump()`.
- Docs: `export_trace()` and the `.ftt` time travel format; `$result1` /
  `$result2` after `export_trace()` (the sampled intervals and those whose
  thread order is approximate).

## [1.0.0] - 2026-10-29

First standalone release of pybridge (previously shipped only inside
[Fathom](https://github.com/EGKrb/Fathom)).

- x64dbg plugin (`pybridge.dp64` / `pybridge.dp32`): local TCP server, JSON
  line protocol, commands `pybridge.start`, `pybridge.stop`, `pybridge.status`,
  `pybridge.emu`.
- Standard-library Python client `x64dbg_bridge.Debugger`: session, execution,
  registers, memory, breakpoints, API hooks, anti-anti-debug, traces
  (`export_trace()` supports JSON, CSV, Tenet and Frinet with Fathom).
- Example scripts, including Unicorn-based minidump emulation.
- PowerShell launcher and twelve templates for `headless.exe`.

[1.1.0]: https://github.com/EGKrb/pybridge/releases/tag/v1.1.0
[1.0.0]: https://github.com/EGKrb/pybridge/releases/tag/v1.0.0
