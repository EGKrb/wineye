# Security Policy

## Supported versions

Fixes go into the latest release only (see [CHANGELOG.md](CHANGELOG.md)). The copies of `wineye.py` shipped in [Fathom](https://github.com/EGKrb/Fathom) and [HvKdFusion](https://github.com/EGKrb/HvKdFusion) are kept identical to the latest release.

| Version | Supported |
| ------- | --------- |
| 1.2.x   | ✅ |
| < 1.2   | ❌ (update to 1.2.x) |

## Scope and intended use

**wineye** is a WinDbg-style memory inspector and crash-dump debugger for Windows targets: kernel crash dumps, user-mode minidumps, raw memory images and live Hyper-V VMs. It is intended for **legitimate, authorized work**: incident response and memory forensics, malware analysis in a controlled environment, debugging your own systems and drivers, security research and CTF challenges. Only inspect machines, VMs and memory images you are allowed to analyze — a memory image holds everything that was in RAM, including credentials and personal data.

## A note on the attack surface

- **wineye parses untrusted input**: dumps and memory images, the module images and PDBs found in them or fetched for them, and the guest's own kernel structures. A crafted file or a hostile guest can make it read wrong data. Memory is only ever read from the target, never executed, and wineye loads no code from it.
- **Symbols** are downloaded over HTTPS from Microsoft's public symbol server (`https://msdl.microsoft.com/download/symbols`) into a local cache (`~/.wineye/symbols`), or read from the paths you give with `--sympath`. Only point `--sympath` at folders you trust.
- **Live VMs** (`--vm`, `--from-vm`) need **administrator** rights and the Hyper-V components set up by HvKdFusion (`hvlib.dll` / `hvmm.sys`, MemProcFS / LeechCore). wineye's own access is read-only; it can freeze and resume the VM on request.
- **Server modes** (`--serve mcp|dap|gdb`) expose the target's memory, read-only, over stdio or TCP. TCP listeners bind to `127.0.0.1` by default: any local program that connects can read the target's memory, so stop the server when you don't need it, and don't bind it to another interface.
- **`.dump /u`** writes a minidump with the memory of a process to the path you give.

## Reporting a vulnerability

If you find a security issue in `wineye.py` — for example a crash, hang or runaway memory use on a crafted dump, image or PDB, a path-handling bug in the symbol cache, or a server accepting input in an unsafe way:

1. **Do not** open a public issue describing the exploit.
2. Report it privately through [GitHub's "Report a vulnerability"](https://github.com/EGKrb/wineye/security/advisories/new) (Security tab → Advisories).
3. Please include the wineye release (or the commit of `wineye.py`), the kind of target, a description and reproduction steps. Don't attach real memory images: they contain sensitive data.

Vulnerabilities in **MemProcFS / LeechCore**, **LiveCloudKd (hvlib / hvmm)** or Python itself should be reported to their upstream projects.

There is no formal SLA for this project, but reports are read and taken seriously.
