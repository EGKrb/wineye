# Changelog

## [1.0.0] - 2026-10-29

First standalone release of wineye (previously shipped only inside
[HvKdFusion](https://github.com/EGKrb/HvKdFusion) and [Fathom](https://github.com/EGKrb/Fathom)).

- WinDbg-style REPL on passive memory through MemProcFS's `vmm.dll`
  (`ps`, `!process`, `.pid`, `lm`, `x`, `db/dw/dd/dq`, `dt`, `!vad`, `!handle`, `s`).
- Targets: kernel crash dumps, raw memory images, any LeechCore device, live
  Hyper-V VMs (`hvmm`), with the hvlib control channel (`freeze`, `resume`, `r`).
- Scripting (`-c`, `--batch`, `--json`) and a `Vmm` Python class.
- Server modes `--serve mcp | gdb | dap`.

[1.0.0]: https://github.com/EGKrb/wineye/releases/tag/v1.0.0
