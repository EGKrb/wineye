# Changelog

## [1.1.0] - 2026-10-30

Crash-dump debugger: wineye now reads PDBs, unwinds x64 stacks and disassembles
by itself (still one Python file, standard library only).

- `!analyze [-v]`: bugcheck name and arguments, faulting context (trap frame /
  context record from the arguments), faulting instruction, module / driver,
  symbol, stack, failure bucket.
- Call stacks `k`, `!thread`, `!process <pid> 7`: x64 unwinding from `.pdata`,
  kernel to user mode across the system call (and interrupts), waiting threads
  from their switch frame, session space (win32k).
- Contexts: `~` / `~<n>` processors, `.thread`, `.cxr`, `.trap`, `r`, `.frame`.
- Disassembly: `u`, `uf` (whole function, split chunks) in WinDbg syntax, `ln`.
- Private PDBs (`--sympath`, `.sympath`, `WINEYE_SYMPATH`): source lines in stacks,
  `dv` (parameters and locals with their values); Microsoft symbols and missing
  module images downloaded into `~/.wineye/symbols` (`--no-symbol-download`).
- `.dump /u [pid] <file>`: user-mode minidump of a process (threads, modules,
  resident memory) for WinDbg / Visual Studio.
- Native engine (`--engine native`, automatic fallback for `.dmp`): every Windows
  x64 kernel dump format without MemProcFS.
- Addresses are expressions, hexadecimal by default as in WinDbg.
- MCP tools `stack`, `analyze`, `disassemble`; DAP `stackTrace` returns real frames.
- Fixed: `vmm.dll` never initialized (its argument parsing starts at index 0);
  kernel reads in a KVA-shadowed process context now fall back to System.
- User-mode minidumps (`MDMP`: WinDbg `.dump`, Task Manager, `MiniDumpWriteDump`,
  `.dump /u`) open with their own engine: `~` lists the threads, `!analyze -v` reads
  the exception record, `k`, `dv`, `u`, `db`... in user mode.
- Kernel dumps without user pages: modules rebuilt from the VADs (image section:
  name, SizeOfImage, CheckSum, file size, TimeDateStamp from Windows 10 2004), so the
  user-mode frame where each thread entered the kernel is symbolized; on older builds
  the image is recognized locally by SizeOfImage + CheckSum + file size. `k` says
  where the user stack stops; `.dump /u` writes threads + modules without memory.
- `kb` (parameter home slots, "Args to Child"); `dv` without a private PDB shows them,
  marked as unreliable; with one, structures field by field and `char *` strings.
- Tests: `tests/test_debugger.py` (disassembler, PDB fixture, unwinder, synthetic
  crash dump, VAD modules, local images, synthetic user minidump), run on Windows
  and Linux. `dv` was also checked on a real crash minidump of a `/Zi /Od` program.

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
[1.1.0]: https://github.com/EGKrb/wineye/releases/tag/v1.1.0
