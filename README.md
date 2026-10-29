# wineye

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![Platform: Windows](https://img.shields.io/badge/Platform-Windows%20x64-0078D6.svg)](#) [![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB.svg)](#) [![Tests](https://github.com/EGKrb/wineye/actions/workflows/tests.yml/badge.svg)](https://github.com/EGKrb/wineye/actions/workflows/tests.yml)

**wineye** is a native-Windows, **WinDbg-style memory inspector**: it looks at a Windows target from its memory alone — a kernel crash dump, a raw memory image, or a **live Hyper-V VM** — with no debug transport in the guest and the target unaware it is being inspected. It drives [MemProcFS](https://github.com/ufrisk/MemProcFS)'s `vmm.dll` directly through `ctypes`: **one Python file, no WSL, no pip package, no Python extension**.

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
```

---

## Install

1. **Python 3.10+** (standard library only).
2. **`vmm.dll`** from a [MemProcFS release](https://github.com/ufrisk/MemProcFS/releases) (x64), with `leechcore.dll` next to it.
3. Download [`wineye.py`](wineye.py) (or the [release zip](https://github.com/EGKrb/wineye/releases/latest)).

```powershell
python wineye.py --vmm C:\Tools\MemProcFS\vmm.dll --dump C:\dumps\MEMORY.DMP
```

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

Addresses accept `0x..`, decimal and `module!symbol`.

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
| **MCP** | `wineye.py --dump d.raw --serve mcp` | MCP server (JSON-RPC 2.0 over stdio) for an AI/agent client. Tools: `list_processes`, `process_info`, `list_modules`, `list_handles`, `list_vads`, `read_memory`, `resolve_symbol`, `registers`. |
| **gdbserver** | `wineye.py --dump d.raw --serve gdb [--listen 127.0.0.1:2159]` | GDB Remote Serial Protocol over TCP: `target remote 127.0.0.1:2159`, then `x/16xg <addr>` and, on a live VM, `info registers`. |
| **DAP** | `wineye.py --dump d.raw --serve dap [--listen 127.0.0.1:4711]` | Debug Adapter Protocol (stdio, or TCP with `--listen`) for an IDE: `modules`, `readMemory`, `evaluate module!symbol`. |

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
```

Protocol-level tests of the MCP, DAP and gdbserver modes against a fake `Vmm`: no `vmm.dll` and no target needed, so they run anywhere (they run on every push in GitHub Actions).

## Part of

wineye ships with [HvKdFusion](https://github.com/EGKrb/HvKdFusion) (Hyper-V kernel debugging and memory forensics) and [Fathom](https://github.com/EGKrb/Fathom) (reverse-engineering toolkit built on x64dbg), where the `wineye` command and `.init MEMORY.DMP` open it from the debugger. This repository is its standalone home.

## License

GPL-3.0-or-later — see [LICENSE](LICENSE). wineye loads MemProcFS / LeechCore (AGPL-3.0) at run time; they are not included.
