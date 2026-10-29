# wineye

[![Licence : GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![Plateforme : Windows](https://img.shields.io/badge/Plateforme-Windows%20x64-0078D6.svg)](#) [![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB.svg)](#) [![Tests](https://github.com/EGKrb/wineye/actions/workflows/tests.yml/badge.svg)](https://github.com/EGKrb/wineye/actions/workflows/tests.yml)

**wineye** est un **inspecteur mémoire façon WinDbg**, natif Windows : il analyse une cible Windows à partir de sa seule mémoire — dump noyau, image mémoire brute ou **VM Hyper-V en direct** — sans transport de débogage dans l'invité, et sans que la cible le sache. Il pilote directement `vmm.dll` de [MemProcFS](https://github.com/ufrisk/MemProcFS) via `ctypes` : **un seul fichier Python, ni WSL, ni paquet pip, ni extension Python**.

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
```

---

## Installation

1. **Python 3.10+** (bibliothèque standard uniquement).
2. **`vmm.dll`** d'une [release MemProcFS](https://github.com/ufrisk/MemProcFS/releases) (x64), avec `leechcore.dll` à côté.
3. Télécharger [`wineye.py`](wineye.py) (ou le [zip de la release](https://github.com/EGKrb/wineye/releases/latest)).

```powershell
python wineye.py --vmm C:\Tools\MemProcFS\vmm.dll --dump C:\dumps\MEMORY.DMP
```

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

Les adresses acceptent `0x..`, le décimal et `module!symbole`.

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
| **MCP** | `wineye.py --dump d.raw --serve mcp` | serveur MCP (JSON-RPC 2.0 sur stdio) pour un client IA / agent. Outils : `list_processes`, `process_info`, `list_modules`, `list_handles`, `list_vads`, `read_memory`, `resolve_symbol`, `registers`. |
| **gdbserver** | `wineye.py --dump d.raw --serve gdb [--listen 127.0.0.1:2159]` | GDB Remote Serial Protocol sur TCP : `target remote 127.0.0.1:2159`, puis `x/16xg <adr>` et, sur une VM en direct, `info registers`. |
| **DAP** | `wineye.py --dump d.raw --serve dap [--listen 127.0.0.1:4711]` | Debug Adapter Protocol (stdio, ou TCP avec `--listen`) pour un IDE : `modules`, `readMemory`, `evaluate module!symbole`. |

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
```

Tests au niveau protocole des modes MCP, DAP et gdbserver, avec un `Vmm` factice : ni `vmm.dll` ni cible nécessaires, ils tournent partout (à chaque push dans GitHub Actions).

## Intégré à

wineye est livré avec [HvKdFusion](https://github.com/EGKrb/HvKdFusion) (débogage noyau et forensic mémoire Hyper-V) et [Fathom](https://github.com/EGKrb/Fathom) (boîte à outils de rétro-ingénierie construite sur x64dbg), où la commande `wineye` et `.init MEMORY.DMP` l'ouvrent depuis le débogueur. Ce dépôt est sa version autonome.

## Licence

GPL-3.0-or-later — voir [LICENSE](LICENSE). wineye charge MemProcFS / LeechCore (AGPL-3.0) à l'exécution ; ils ne sont pas inclus.
