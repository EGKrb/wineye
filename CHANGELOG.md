# Changelog

## [1.2.0] - 2026-10-30

32-bit code of WoW64 processes is unwound (it was x64 only), with or without PDBs.

- x86 unwinder: FPO / FrameData program strings of the module's PDB (as WinDbg,
  with the `.raSearch` alignment scan), old FPO records.
- Without unwind data, the code itself: the stack layout of the function from its
  start to the instruction (push / pop / `sub esp` / ebp frame / `enter` / `leave`,
  calls with their `ret N`, `__chkstk`), giving the return address slot and the saved
  registers, even in `/Oy` code. The start comes from the symbols, from a return
  address on the stack whose call targets it (accepted only if the layout lands on
  that slot), or from the `int 3` padding before it. The ebp chain bounds it and is
  the fallback; a frame found by a plain stack scan is marked as uncertain. On real
  dumps (`/O2` and `/Od`), stacks without any PDB equal those with PDBs.
- 32-bit disassembler mode (`X64Disassembler.decode(..., mode32=True)`).
- WoW64 threads: the 32-bit frames are inserted between the wow64cpu entry and the
  CPU simulation loop, from the WOW64_CONTEXT saved in the TEB's CPU area
  (`TlsSlots[WOW64_TLS_CPURESERVED]`); a thread running 32-bit code (cs 0x23) is
  unwound from its own registers. `k` marks the 64-bit / 32-bit transitions.
- On x86 frames: `kb` shows the first 3 stack parameters, `dv` reads `S_BPREL32`
  (x86 `/Od`) and FPO variables (relative to the function entry), `r` shows the
  32-bit registers, `eax`...`eip` work in expressions; `u`, `uf` (bounded by the
  symbol, no `.pdata`) and `!analyze` disassemble 32-bit code.
- x86 minidumps (written by a 32-bit tool) open; 64-bit minidumps of WoW64
  processes say so, and `k` says where the 32-bit frames would be when the source
  does not hold the TEB / CPU area (small minidumps).
- Native engine: the 32-bit modules of WoW64 processes (PEB32 loader list).
- Modules whose loader strings or PE header are paged out are completed from the
  VADs (name, TimeDateStamp, CheckSum, file size), to fetch their image and PDB.
- MemProcFS: `vmm.dll` fills the pages it cannot read with zeros; wineye now takes
  such metadata pages as missing (image from the symbol server instead). The 64-bit
  ntdll of a WoW64 process (`_64-ntdll.dll`) is named `ntdll`.
- Fixed: in a minidump whose thread stack ranges lie inside its memory list (wineye's
  `.dump /u`), the memory just past a stack range was unreadable (WoW64 CPU area).
- x86 public names shown undecorated (`_NtDelayExecution@8` -> `NtDelayExecution`).
- Fixed: export directory and `.pdata` read at PE32+ offsets in 32-bit images.
- A local copy of a module image (same path, or in the search path) is used when its
  TimeDateStamp and SizeOfImage match and the symbol server does not have it (small
  minidumps of your own programs).
- Tests: `tests/data/fixture32.exe` / `.pdb` (x86, FPO + ebp frames), WoW64, x86
  without PDB (layout, call check, padding, ebp bound), x86 minidump, PEB32, VAD
  completion, zero-filled pages, overlapping minidump ranges. Checked on real dumps:
  64-bit minidumps of WoW64 processes (`/O2`, `/Od`, with and without PDBs), x86 crash
  minidumps, a Windows 10 kernel + user dump with 4 WoW64 processes (native engine,
  and MemProcFS on its physical image) and a `.dump /u` of it; x64 output unchanged on
  4 other kernel dumps.

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
