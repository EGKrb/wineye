# wineye

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![Platform: Windows](https://img.shields.io/badge/Platform-Windows%20x64-0078D6.svg)](#) [![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB.svg)](#) [![Tests](https://github.com/EGKrb/wineye/actions/workflows/tests.yml/badge.svg)](https://github.com/EGKrb/wineye/actions/workflows/tests.yml)

**wineye** is a native-Windows, **WinDbg-style memory inspector**: it looks at a Windows target from its memory alone — a kernel crash dump, a raw memory image, or a **live Hyper-V VM** — with no debug transport in the guest and the target unaware it is being inspected. It drives [MemProcFS](https://github.com/ufrisk/MemProcFS)'s `vmm.dll` directly through `ctypes`, or reads Windows kernel crash dumps itself: **one Python file, no WSL, no pip package, no Python extension**.

It is also a **crash-dump debugger**: `!analyze -v`, call stacks from kernel mode down into user mode, trap frames and context records, processors, disassembly, source lines and local variables with private PDBs, and user-mode minidump extraction (`.dump /u`) — with its own PDB reader, x64 stack unwinder and x86-64 disassembler.

It can also run as an **MCP server** (for AI/agent clients), a **gdbserver** or a **Debug Adapter Protocol** server.

🌐 **Website:** <https://egkrb.github.io/wineye/> · 🇫🇷 [Lire en français](README.fr.md) · [⬇ Latest release](https://github.com/EGKrb/wineye/releases/latest)

```
wineye> ps
   PID   PPID            EPROCESS                 PEB  Name
----------------------------------------------------------------------
     4      0  0xffffb00012340004  0x0000000000000000  System
  7480   6312  0xffffb00012347480  0x0000007ff0000000  notepad.exe
wineye> .pid 7480
wineye> lm
wineye> x nt!PsInitialSystemProcess
wineye> dq nt!PsInitialSystemProcess L4
wineye> !thread ffffc00c518a4080
THREAD 0xffffc00c518a4080  Cid 0xb20.0xb24  Teb 0x942000  processus explorer.exe (pid 2848)
  Child-SP          RetAddr           Site d'appel
  fffff40c99aeeeb0  fffff803f2ae5d76  nt!KiSwapContext+0x76
  ...
  fffff40c99aef480  fffff803f2c3ef13  win32kfull!NtUserWaitMessage+0x42
  fffff40c99aef4c0  00007ffab2a71204  nt!KiSystemServiceCopyEnd+0x13
      --- passage en mode utilisateur ---
  00000000006ef6c8  00007ffab4478b8d  win32u!ZwUserWaitMessage+0x14
  00000000006ef6d0  00007ffab4743356  SHELL32!CDesktopBrowser::_MessageLoop+0x311
  ...
  00000000006efc90  0000000000000000  ntdll!RtlUserThreadStart+0x21
```

(wineye's messages are in French.)

---

## Install

1. **Python 3.10+** (standard library only).
2. Download [`wineye.py`](wineye.py) (or the [release zip](https://github.com/EGKrb/wineye/releases/latest)).
3. For raw images, LeechCore devices and live VMs: **`vmm.dll`** from a [MemProcFS release](https://github.com/ufrisk/MemProcFS/releases) (x64), with `leechcore.dll` next to it. Kernel crash dumps (`.dmp`) also open **without MemProcFS**, with the native engine.

```powershell
python wineye.py --dump C:\dumps\MEMORY.DMP                                   # native engine
python wineye.py --vmm C:\Tools\MemProcFS\vmm.dll --dump C:\dumps\MEMORY.DMP   # MemProcFS
```

| Option | Effect |
|---|---|
| `--engine auto\|vmm\|native` | `vmm` = MemProcFS; `native` = wineye's own reader of Windows x64 kernel dumps (full, bitmap, kernel, kernel + user, complete, live kernel); `auto` (default) = `vmm.dll`, else native for a `.dmp` |
| `--sympath <dir;dir>` | folders of private PDBs (repeatable; also `WINEYE_SYMPATH`) |
| `--no-symbol-download` | stay offline (otherwise Microsoft PDBs, and module images missing from the dump, are downloaded into `~/.wineye/symbols`) |

Instead of `--vmm`, a `config.json` next to `wineye.py` can point to MemProcFS (`"memprocfs_exe"` or `"leechcore_dll"`; `vmm.dll` is looked up in the same folder) — this is the file [HvKdFusion](https://github.com/EGKrb/HvKdFusion) writes with `hvkdfusion.py setup`.

## Targets

| Option | Target |
|---|---|
| `--dump <file>` | kernel crash dump (`.dmp`) or raw memory image (`.raw`, `.mem`, `.vmem`) |
| `--device <dev>` | any LeechCore device (`fpga`, `pmem`, `hvmm`, ...) |
| `--live` | live Hyper-V VM (LeechCore `hvmm` device, from LiveCloudKd) |
| `--from-vm <N>` | linear dump of Hyper-V VM *N*, then analysis (needs HvKdFusion) |
| `--vm <N>` | adds the hvlib **control channel** on live VM *N*: `freeze`, `resume`/`g`, `r` (registers) |

Live VMs and `--from-vm` / `--vm` need administrator rights, Hyper-V and the components that [HvKdFusion](https://github.com/EGKrb/HvKdFusion) sets up (`hvkdfusion.py` next to `wineye.py`). Dumps and images need only `vmm.dll`.

## Commands

| Command | Effect |
|---|---|
| `ps` | processes |
| `!process [pid]` | process details (default: current context) |
| `.pid <n\|-1>` | process context (`-1` = physical memory) |
| `lm [pid]` | modules |
| `x <mod>!<sym>` | resolve a symbol through the PDB (`x nt!PsInitialSystemProcess`) |
| `db\|dw\|dd\|dq <addr> [L<n>]` | hex dump (bytes / words / dwords / qwords) |
| `dt <mod>!<type> [field] [addr]` | type size, field offset (and its value at `addr`) |
| `!vad [pid]`, `!handle [pid]` | VADs, handles |
| `s <addr> <len> <hex>` | byte pattern search |
| `.ctl <N>`, `freeze`, `resume`/`g`, `r [reg...]` | control channel on a live VM |
| `info`, `help`, `q` | summary, help, quit |

Addresses are expressions, **hexadecimal by default** as in WinDbg (`fffff807` + backtick + `50001000`, `0x..` too; `0n` for decimal), with registers of the current context (`rsp`, `@rip`), `module!symbol`, `+` and `-`. Counts (`L<n>`) are decimal, or `0x..`.

## Debugger

On crash dumps, images and live VMs — with Microsoft's symbols (downloaded) or your private PDBs (`--sympath`):

| Command | Effect |
|---|---|
| `!analyze [-v]` | bugcheck analysis: stop code, name and arguments, faulting context (trap frame / context record taken from the arguments), faulting instruction, **module / driver**, symbol, stack. On a dump that is not a crash: the current processor's context |
| `k [n]` | call stack of the current context, **kernel then user mode across the system call** (and interrupts / machine frames); source lines with a private PDB |
| `~`, `~<n>` | processors (current thread, process, rip); switch to processor *n* |
| `!thread [ethread]` | thread details and stack (waiting threads are unwound from their switch frame) |
| `.thread <ethread>` | switch the context to a thread |
| `.cxr [addr]` | context from a `CONTEXT` record (no argument: back to the dump's context) |
| `.trap <addr>` | context from a `KTRAP_FRAME` |
| `r [reg...]` | registers of the current context / selected frame |
| `u [addr] [L<n>]`, `uf <func>` | disassembly (WinDbg syntax, symbolized targets); `uf` follows the whole function, split chunks included |
| `ln <addr>` | nearest symbol and source line |
| `.frame [n]`, `dv` | select a frame; its **parameters and locals** with their values (private PDB) |
| `!process <pid> 7` | a process with its threads and their stacks |
| `.dump /u [pid] <file>` | write a **user-mode minidump** of a process (threads with their user-mode registers, modules, every resident page), readable by WinDbg / Visual Studio |
| `.sympath [p;p]`, `.reload` | private PDB path; reload symbols |

```
wineye> !analyze -v
SYSTEM_SERVICE_EXCEPTION (3b)
Arg1: 00000000c0000005, code d'exception (violation d'accès)
Arg3: fffff803f1e9a370, enregistrement de contexte (CONTEXT)
...
CONTEXTE : .cxr 0xfffff803f1e9a370
FAULTING_IP : nt!DebugService2+0x6
STACK_TEXT : ...
MODULE_NAME : vmbus
IMAGE_NAME : vmbus.sys
SYMBOL_NAME : vmbus!XPartEnlightenedIsr
FAILURE_BUCKET_ID : SYSTEM_SERVICE_EXCEPTION_vmbus!XPartEnlightenedIsr
wineye> .dump /u 2848 C:\dumps\explorer.dmp
[+] C:\dumps\explorer.dmp : 90 threads, 202 modules, 1679 plages, 24.5 Mo de mémoire
```

What it needs from the target: a *kernel* dump holds no user-mode pages (*kernel + user*, *complete* and full dumps do), so user-mode frames and `.dump /u` need one of those; `dv` needs a private PDB (Microsoft's public PDBs have no locals).

**Scripting:** `-c "ps;lm"` runs commands (repeatable), `--batch` quits afterwards, `--json` prints structured output:

```powershell
python wineye.py --dump vm0.raw -c "ps;lm 4" --batch --json
```

From Python, import it and use the `Vmm` class:

```python
import wineye
vmm = wineye.Vmm(r"C:\Tools\MemProcFS\vmm.dll", ["-device", r"C:\dumps\vm0.raw", "-waitinitialize"])
for pid in vmm.pids():
    print(vmm.process(pid)["name"])
vmm.close()
```

## Server modes (`--serve`)

Instead of the REPL, wineye can serve the same target to other tools. All modes are **read-only passive memory** (VP registers only on a live VM with `--vm N`), with no breakpoints or stepping.

| Mode | Command | Use |
|---|---|---|
| **MCP** | `wineye.py --dump d.raw --serve mcp` | MCP server (JSON-RPC 2.0 over stdio) for an AI/agent client. Tools: `list_processes`, `process_info`, `list_modules`, `list_handles`, `list_vads`, `read_memory`, `resolve_symbol`, `registers`, `stack`, `analyze`, `disassemble`. |
| **gdbserver** | `wineye.py --dump d.raw --serve gdb [--listen 127.0.0.1:2159]` | GDB Remote Serial Protocol over TCP: `target remote 127.0.0.1:2159`, then `x/16xg <addr>` and, on a live VM, `info registers`. |
| **DAP** | `wineye.py --dump d.raw --serve dap [--listen 127.0.0.1:4711]` | Debug Adapter Protocol (stdio, or TCP with `--listen`) for an IDE: `modules`, `stackTrace` (real frames), `readMemory`, `evaluate module!symbol`. |

Register wineye as an MCP server, for example in an MCP client configuration:

```json
{ "command": "python",
  "args": ["C:\\Tools\\wineye\\wineye.py", "--vmm", "C:\\Tools\\MemProcFS\\vmm.dll",
           "--dump", "C:\\dumps\\vm0.raw", "--serve", "mcp"] }
```

In every server mode the startup banner goes to stderr, so stdout carries only the protocol.

## What wineye is not

Passive memory gives **no run control**: no breakpoints, no single-stepping, no register writes (the same limits as [ntoseye](https://github.com/dmaivel/ntoseye)'s memory backend). The live-VM control channel adds freeze/resume and register reads, but hvlib delivers no trap events. For run control, use WinDbg/kd (`hvkdfusion.py debug` in HvKdFusion).

## Tests

```powershell
python tests\test_servers.py
python tests\test_debugger.py
```

- `test_servers.py`: protocol-level tests of the MCP, DAP and gdbserver modes against a fake `Vmm`.
- `test_debugger.py`: disassembler vectors, `CONTEXT` records, the PDB reader on a small private PDB (`tests/data/fixture.pdb`, built from `fixture.c`: lines, parameters, locals, types), x64 unwinding and `k` / `dv` / `.frame` / `u` / `uf` on a synthetic process, and the crash-dump reader on a synthetic `PAGEDU64` dump.

No `vmm.dll` and no target needed, so they run anywhere (on every push in GitHub Actions, on Windows and Linux).

## Part of

wineye ships with [HvKdFusion](https://github.com/EGKrb/HvKdFusion) (Hyper-V kernel debugging and memory forensics) and [Fathom](https://github.com/EGKrb/Fathom) (reverse-engineering toolkit built on x64dbg), where the `wineye` command and `.init MEMORY.DMP` open it from the debugger. This repository is its standalone home.

## License

GPL-3.0-or-later — see [LICENSE](LICENSE). wineye loads MemProcFS / LeechCore (AGPL-3.0) at run time; they are not included. Symbols and module images come from Microsoft's public symbol server at run time and are not included either.
