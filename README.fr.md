# wineye

[![Licence : GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![Plateforme : Windows](https://img.shields.io/badge/Plateforme-Windows%20x64-0078D6.svg)](#) [![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB.svg)](#) [![Tests](https://github.com/EGKrb/wineye/actions/workflows/tests.yml/badge.svg)](https://github.com/EGKrb/wineye/actions/workflows/tests.yml)

**wineye** est un **inspecteur mémoire façon WinDbg**, natif Windows : il analyse une cible Windows à partir de sa seule mémoire — dump noyau, image mémoire brute ou **VM Hyper-V en direct** — sans transport de débogage dans l'invité, et sans que la cible le sache. Il pilote directement `vmm.dll` de [MemProcFS](https://github.com/ufrisk/MemProcFS) via `ctypes`, ou lit lui-même les dumps noyau Windows : **un seul fichier Python, ni WSL, ni paquet pip, ni extension Python**.

C'est aussi un **débogueur de dumps** : `!analyze -v`, piles d'appels du noyau jusqu'en mode utilisateur, trap frames et enregistrements de contexte, processeurs, désassemblage, lignes source et variables locales avec des PDB privés, et extraction de minidumps user-mode (`.dump /u`) — avec son propre lecteur de PDB, son dérouleur de pile x64 et son désassembleur x86-64.

Il peut aussi tourner en **serveur MCP** (pour un client IA / agent), en **gdbserver** ou en serveur **Debug Adapter Protocol**.

🌐 **Site :** <https://egkrb.github.io/wineye/> · 🇬🇧 [English](README.md) · [⬇ Dernière release](https://github.com/EGKrb/wineye/releases/latest)

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

---

## Installation

1. **Python 3.10+** (bibliothèque standard uniquement).
2. Télécharger [`wineye.py`](wineye.py) (ou le [zip de la release](https://github.com/EGKrb/wineye/releases/latest)).
3. Pour les images brutes, les devices LeechCore et les VM en direct : **`vmm.dll`** d'une [release MemProcFS](https://github.com/ufrisk/MemProcFS/releases) (x64), avec `leechcore.dll` à côté. Les dumps noyau (`.dmp`) s'ouvrent aussi **sans MemProcFS**, avec le moteur natif.

```powershell
python wineye.py --dump C:\dumps\MEMORY.DMP                                   # moteur natif
python wineye.py --vmm C:\Tools\MemProcFS\vmm.dll --dump C:\dumps\MEMORY.DMP   # MemProcFS
```

| Option | Effet |
|---|---|
| `--engine auto\|vmm\|native` | `vmm` = MemProcFS ; `native` = lecteur intégré des dumps noyau Windows x64 (complet, bitmap, noyau, noyau + user, mémoire complète, live) ; `auto` (défaut) = `vmm.dll`, sinon natif pour un `.dmp` |
| `--sympath <dossier;dossier>` | dossiers de PDB privés (répétable ; aussi `WINEYE_SYMPATH`) |
| `--no-symbol-download` | hors ligne (sinon les PDB Microsoft, et les images de modules absentes du dump, sont téléchargés dans `~/.wineye/symbols`) |

À la place de `--vmm`, un `config.json` à côté de `wineye.py` peut indiquer MemProcFS (`"memprocfs_exe"` ou `"leechcore_dll"` ; `vmm.dll` est cherché dans le même dossier) — c'est le fichier qu'écrit [HvKdFusion](https://github.com/EGKrb/HvKdFusion) avec `hvkdfusion.py setup`.

## Cibles

| Option | Cible |
|---|---|
| `--dump <fichier>` | dump noyau (`.dmp`) ou image mémoire brute (`.raw`, `.mem`, `.vmem`) |
| `--device <dev>` | n'importe quel device LeechCore (`fpga`, `pmem`, `hvmm`...) |
| `--live` | VM Hyper-V en direct (device LeechCore `hvmm`, issu de LiveCloudKd) |
| `--from-vm <N>` | dump linéaire de la VM Hyper-V *N*, puis analyse (nécessite HvKdFusion) |
| `--vm <N>` | ajoute le **canal de contrôle** hvlib sur la VM *N* en direct : `freeze`, `resume`/`g`, `r` (registres) |

Les VM en direct et `--from-vm` / `--vm` demandent les droits administrateur, Hyper-V et les composants installés par [HvKdFusion](https://github.com/EGKrb/HvKdFusion) (`hvkdfusion.py` à côté de `wineye.py`). Les dumps et images n'ont besoin que de `vmm.dll`.

## Commandes

| Commande | Effet |
|---|---|
| `ps` | processus |
| `!process [pid]` | détails d'un processus (par défaut : contexte courant) |
| `.pid <n\|-1>` | contexte de processus (`-1` = mémoire physique) |
| `lm [pid]` | modules |
| `x <mod>!<sym>` | résout un symbole via le PDB (`x nt!PsInitialSystemProcess`) |
| `db\|dw\|dd\|dq <adr> [L<n>]` | dump hexadécimal (octets / mots / dwords / qwords) |
| `dt <mod>!<type> [champ] [adr]` | taille d'un type, offset d'un champ (et sa valeur à `adr`) |
| `!vad [pid]`, `!handle [pid]` | VAD, handles |
| `s <adr> <long> <hex>` | recherche d'un motif d'octets |
| `.ctl <N>`, `freeze`, `resume`/`g`, `r [reg...]` | canal de contrôle sur une VM en direct |
| `info`, `help`, `q` | résumé, aide, quitter |

Les adresses sont des expressions, **hexadécimales par défaut** comme dans WinDbg (`fffff807` + accent grave + `50001000`, `0x..` aussi ; `0n` pour le décimal), avec les registres du contexte courant (`rsp`, `@rip`), `module!symbole`, `+` et `-`. Les nombres (`L<n>`) sont décimaux, ou `0x..`.

## Débogueur

Sur les dumps, les images et les VM en direct — avec les symboles Microsoft (téléchargés) ou vos PDB privés (`--sympath`) :

| Commande | Effet |
|---|---|
| `!analyze [-v]` | analyse du bugcheck : code d'arrêt, nom et arguments, contexte fautif (trap frame / enregistrement de contexte tirés des arguments), instruction fautive, **module / driver**, symbole, pile. Sur un dump qui n'est pas un plantage : le contexte du processeur courant |
| `k [n]` | pile d'appels du contexte courant, **noyau puis mode utilisateur à travers l'appel système** (et les interruptions / machine frames) ; lignes source avec un PDB privé |
| `~`, `~<n>` | processeurs (thread courant, processus, rip) ; bascule sur le processeur *n* |
| `!thread [ethread]` | détails et pile d'un thread (les threads en attente sont déroulés depuis leur switch frame) |
| `.thread <ethread>` | bascule le contexte sur un thread |
| `.cxr [adr]` | contexte depuis un enregistrement `CONTEXT` (sans argument : retour au contexte du dump) |
| `.trap <adr>` | contexte depuis un `KTRAP_FRAME` |
| `r [reg...]` | registres du contexte courant / de la frame sélectionnée |
| `u [adr] [L<n>]`, `uf <fonction>` | désassemblage (syntaxe WinDbg, cibles symbolisées) ; `uf` suit toute la fonction, morceaux séparés compris |
| `ln <adr>` | symbole et ligne source les plus proches |
| `.frame [n]`, `dv` | sélectionne une frame ; ses **paramètres et variables locales** avec leurs valeurs (PDB privé) |
| `!process <pid> 7` | un processus avec ses threads et leurs piles |
| `.dump /u [pid] <fichier>` | écrit un **minidump user-mode** d'un processus (threads avec leurs registres user-mode, modules, toutes les pages résidentes), lisible par WinDbg / Visual Studio |
| `.sympath [p;p]`, `.reload` | chemin des PDB privés ; recharge les symboles |

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

Ce qu'il faut dans la cible : un dump *noyau* ne contient aucune page user-mode (les dumps *noyau + user*, *mémoire complète* et complets en ont), donc les frames user-mode et `.dump /u` demandent l'un de ceux-là ; `dv` demande un PDB privé (les PDB publics de Microsoft n'ont pas de variables locales).

**Scripts :** `-c "ps;lm"` exécute des commandes (répétable), `--batch` quitte ensuite, `--json` produit une sortie structurée :

```powershell
python wineye.py --dump vm0.raw -c "ps;lm 4" --batch --json
```

Depuis Python, on l'importe et on utilise la classe `Vmm` :

```python
import wineye
vmm = wineye.Vmm(r"C:\Tools\MemProcFS\vmm.dll", ["-device", r"C:\dumps\vm0.raw", "-waitinitialize"])
for pid in vmm.pids():
    print(vmm.process(pid)["name"])
vmm.close()
```

## Modes serveur (`--serve`)

À la place de la REPL, wineye sert la même cible à d'autres outils. Tous les modes sont en **lecture seule sur la mémoire passive** (registres VP uniquement sur une VM en direct avec `--vm N`), sans points d'arrêt ni pas à pas.

| Mode | Commande | Usage |
|---|---|---|
| **MCP** | `wineye.py --dump d.raw --serve mcp` | serveur MCP (JSON-RPC 2.0 sur stdio) pour un client IA / agent. Outils : `list_processes`, `process_info`, `list_modules`, `list_handles`, `list_vads`, `read_memory`, `resolve_symbol`, `registers`, `stack`, `analyze`, `disassemble`. |
| **gdbserver** | `wineye.py --dump d.raw --serve gdb [--listen 127.0.0.1:2159]` | GDB Remote Serial Protocol sur TCP : `target remote 127.0.0.1:2159`, puis `x/16xg <adr>` et, sur une VM en direct, `info registers`. |
| **DAP** | `wineye.py --dump d.raw --serve dap [--listen 127.0.0.1:4711]` | Debug Adapter Protocol (stdio, ou TCP avec `--listen`) pour un IDE : `modules`, `stackTrace` (vraies frames), `readMemory`, `evaluate module!symbole`. |

Déclarer wineye comme serveur MCP, par exemple dans la configuration d'un client MCP :

```json
{ "command": "python",
  "args": ["C:\\Tools\\wineye\\wineye.py", "--vmm", "C:\\Tools\\MemProcFS\\vmm.dll",
           "--dump", "C:\\dumps\\vm0.raw", "--serve", "mcp"] }
```

Dans tous les modes serveur, la bannière de démarrage part sur stderr : stdout ne porte que le protocole.

## Ce que wineye n'est pas

La mémoire passive ne donne **aucun contrôle d'exécution** : pas de points d'arrêt, pas de pas à pas, pas d'écriture de registres (les mêmes limites que le backend mémoire de [ntoseye](https://github.com/dmaivel/ntoseye)). Le canal de contrôle des VM en direct ajoute gel/reprise et lecture des registres, mais hvlib ne remonte aucun événement de trap. Pour le contrôle d'exécution, utiliser WinDbg/kd (`hvkdfusion.py debug` dans HvKdFusion).

## Tests

```powershell
python tests\test_servers.py
python tests\test_debugger.py
```

- `test_servers.py` : tests au niveau protocole des modes MCP, DAP et gdbserver, avec un `Vmm` factice.
- `test_debugger.py` : vecteurs du désassembleur, enregistrements `CONTEXT`, lecteur de PDB sur un petit PDB privé (`tests/data/fixture.pdb`, compilé depuis `fixture.c` : lignes, paramètres, variables locales, types), déroulage x64 et `k` / `dv` / `.frame` / `u` / `uf` sur un processus synthétique, et lecteur de dumps sur un dump `PAGEDU64` synthétique.

Ni `vmm.dll` ni cible nécessaires : ils tournent partout (à chaque push dans GitHub Actions, sous Windows et Linux).

## Intégré à

wineye est livré avec [HvKdFusion](https://github.com/EGKrb/HvKdFusion) (débogage noyau et forensic mémoire Hyper-V) et [Fathom](https://github.com/EGKrb/Fathom) (boîte à outils de rétro-ingénierie construite sur x64dbg), où la commande `wineye` et `.init MEMORY.DMP` l'ouvrent depuis le débogueur. Ce dépôt est sa version autonome.

## Licence

GPL-3.0-or-later — voir [LICENSE](LICENSE). wineye charge MemProcFS / LeechCore (AGPL-3.0) à l'exécution ; ils ne sont pas inclus. Les symboles et les images de modules viennent du serveur de symboles public de Microsoft à l'exécution et ne sont pas inclus non plus.
