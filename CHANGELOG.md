# Changelog

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

[1.0.0]: https://github.com/EGKrb/pybridge/releases/tag/v1.0.0
