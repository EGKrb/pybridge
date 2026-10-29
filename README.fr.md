# pybridge

[![Licence : GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE) [![Plateforme : Windows](https://img.shields.io/badge/Plateforme-Windows%20x86%2Fx64-0078D6.svg)](#) [![Plugin x64dbg](https://img.shields.io/badge/x64dbg-plugin-444.svg)](https://github.com/x64dbg/x64dbg) [![CI](https://github.com/EGKrb/pybridge/actions/workflows/ci.yml/badge.svg)](https://github.com/EGKrb/pybridge/actions/workflows/ci.yml)

**pybridge** permet de **piloter [x64dbg](https://github.com/x64dbg/x64dbg) depuis Python** — l'interface graphique ou `headless.exe` (x64dbg sans fenêtre) : breakpoints, pas à pas, registres, mémoire, hooks d'API, traces. C'est un plugin x64dbg qui ouvre un serveur TCP local, accompagné d'un **client Python en bibliothèque standard uniquement**, de scripts d'exemple et de **modèles prêts à l'emploi pour `headless.exe`**.

🌐 **Site :** <https://egkrb.github.io/pybridge/> · 🇬🇧 [English](README.md) · [⬇ Dernière release](https://github.com/EGKrb/pybridge/releases/latest)

```python
from x64dbg_bridge import Debugger

with Debugger.launch(r"C:\Tools\x64dbg\release") as dbg:      # démarre headless.exe
    dbg.init(r"C:\Windows\System32\whoami.exe")

    # Journaliser chaque chargement de DLL sans arrêter le programme
    dbg.set_breakpoint("kernelbase.LoadLibraryExW", break_=False,
                       log="LoadLibrary({utf16@arg.get(0)})")

    # S'arrêter à la première ouverture de fichier, puis lire les arguments
    dbg.set_breakpoint("kernelbase.CreateFileW")
    if dbg.run(wait=True, timeout=10):
        print("ouverture :", dbg.read_string(dbg.eval("arg.get(0)"), wide=True))
        print(dbg.regs())
```

| Dossier | Contenu |
|---|---|
| [`src/`](src) | le plugin x64dbg (`pybridge.dp64` / `pybridge.dp32`) : un serveur TCP sur **127.0.0.1** qui traduit des requêtes JSON en appels à l'API d'x64dbg |
| [`python/x64dbg_bridge.py`](python/x64dbg_bridge.py) | le client Python (bibliothèque standard uniquement, rien à installer) |
| [`examples/`](examples) | scripts d'exemple : moniteur d'API, export de trace, dépaquetage, diff mémoire, déchiffrement XOR, émulation de dump... |
| [`headless/`](headless) | lanceur PowerShell et modèles de scripts x64dbg pour `headless.exe` |

> **Sécurité** : le serveur n'accepte que les connexions locales, mais tout programme local qui s'y connecte contrôle complètement le débogueur (lecture/écriture mémoire du processus débogué, exécution de commandes). Ne le démarrez que quand vous en avez besoin.

---

## Installation

1. Télécharger `pybridge_<version>.zip` depuis la [dernière release](https://github.com/EGKrb/pybridge/releases/latest).
2. L'extraire **dans le dossier `release` d'x64dbg** (celui qui contient `x64\` et `x32\`). Il ajoute :
   ```
   x64\plugins\pybridge.dp64
   x32\plugins\pybridge.dp32
   pybridge\python\  examples\  headless\
   ```
3. En Python, ajouter `<x64dbg>\release\pybridge\python` à `sys.path` puis `from x64dbg_bridge import Debugger`.

Le serveur démarre :

- automatiquement si la variable d'environnement `X64DBG_PYBRIDGE_PORT` est définie (c'est ce que fait `Debugger.launch()`) ;
- sinon avec la commande `pybridge.start [port]` dans x64dbg (port décimal, 27041 par défaut). Aussi : `pybridge.stop`, `pybridge.status`, `pybridge.emu` (voir [Émuler un dump](#émuler-un-dump-dmp)).

Pour se connecter à un x64dbg déjà ouvert (après `pybridge.start`) : `Debugger()`.

## API Python

Les adresses et valeurs acceptent un entier ou une **expression x64dbg** : `"kernelbase.CreateFileW"`, `"rsp+8"`, `"[rsp]"`, `"mod.base(cible.exe)"`. Dans les expressions, les nombres sont en **hexadécimal** par défaut (`.100` = 100 décimal).

| Catégorie | Méthodes |
|---|---|
| Session | `launch(x64dbg_dir, arch, gui)`, `init(path, arguments, cwd)`, `attach(pid)`, `stop()`, `close()` |
| Exécution | `run(wait, timeout, pass_exceptions)`, `pause()`, `step_into()`, `step_over()`, `step_out()`, `wait(timeout)`, `state()` |
| Registres | `regs()`, `reg(name)`, `set_reg(name, value)` |
| Mémoire | `read(addr, size)`, `write(addr, data)`, `read_ptr(addr)`, `read_string(addr, wide)`, `is_valid(addr)`, `disasm(addr, count)` |
| Breakpoints | `set_breakpoint(addr, condition, log, command, break_, singleshot)`, `delete_breakpoint(addr)`, `set_hardware_breakpoint(addr, kind, size)` |
| Hook d'API | `hook_api(api, log, condition, command, break_)` (log sans pause par défaut), `dump_memory(addr, size, path)` |
| Anti-anti-debug | `hide_debugger()` (commande `hide` d'x64dbg : patche le PEB du processus débogué), `scyllahide_profiles(ini)`, `scyllahide_set_profile(ini, name)` |
| Traces | `trace(record_file, condition, max_steps, step_over)`, `export_trace(trace, output, fmt, slide)` |
| Commandes | `cmd("texte brut")`, `command(name, *args)` (échappement automatique), `eval(expr)`, `result` (`$result`) |

pybridge fonctionne avec **x64dbg officiel**. Une exception : `export_trace()` utilise la commande `TraceExport` (trace → JSON, CSV, Tenet ou Frinet), qui n'existe que dans [Fathom](https://github.com/EGKrb/Fathom), la boîte à outils basée sur x64dbg avec laquelle pybridge est livré.

## Exemples

Dans [`examples/`](examples) (option `--x64dbg <dossier release>` ou variable `X64DBG_DIR` pour choisir l'x64dbg à utiliser) :

| Script | Rôle |
|---|---|
| `api_monitor.py cible.exe` | journalise les appels d'API (CreateFileW, LoadLibraryExW, VirtualProtect...) sans arrêter le programme |
| `trace_to_json.py cible.exe --steps 20000 --out dossier` | enregistre une trace depuis le point d'entrée, l'exporte en JSON/CSV et affiche des statistiques (nécessite Fathom) |
| `dump_memory.py --target cible.exe --addr "mod.base(cible.exe)" --size "mod.size(cible.exe)" --out m.bin` | sauvegarde une zone mémoire (ou `--pid` pour s'attacher) |
| `step_log.py cible.exe --count 50` | exécute pas à pas et affiche les registres modifiés par chaque instruction |
| `antidebug.py cible.exe [--scyllahide-ini ...]` | masque le débogueur (PEB) et journalise les sondes anti-debug sans les bloquer |
| `pe_entropy.py cible.exe [--threshold 7.5]` | entropie des sections PE du processus en cours, dump de celles au-dessus du seuil |
| `memory_diff.py cible.exe --addr ... --size ... --delay 10` | snapshot avant/après d'une plage mémoire, liste les régions modifiées (packers, décompresseurs runtime) |
| `unpack_dump.py cible.exe --out-dir dumps` | dump de chaque région que `VirtualProtect` passe en PAGE_EXECUTE* (noms horodatés + adresse + taille + flag) |
| `xor_decrypt.py cible.exe --addr ... --size ... --key ... [--brute]` | lit une zone chiffrée depuis le processus et la déchiffre en XOR (clé connue ou force brute sur 1 octet) |
| `emulate_dump.py crash.dmp --count 200` | **émule** un dump mémoire (pas à pas, trace JSON/CSV, serveur pybridge) — voir ci-dessous |

## Émuler un dump (.dmp)

Un `.dmp` est une **photo figée** de la mémoire : il ne s'exécute pas, il n'a pas de CPU ni de noyau derrière lui. `examples/emulate_dump.py` charge la mémoire capturée et les registres du thread dans l'émulateur **Unicorn** et déroule le code à partir de là. Le parsing du minidump est en bibliothèque standard ; l'émulation demande `pip install unicorn` (et `capstone` en option, pour le désassemblage).

```powershell
py emulate_dump.py crash.dmp --list-threads            # threads et leur rip
py emulate_dump.py crash.dmp --thread 0 --count 200    # pas à pas émulé
py emulate_dump.py crash.dmp --from "rsp+8"            # démarrer ailleurs (expression sur les registres)
py emulate_dump.py crash.dmp --count 5000 --keep-going --trace t.json --trace-csv t.csv
py emulate_dump.py crash.dmp --serve --port 27041      # piloter via x64dbg_bridge.Debugger
```

- **Mode fidèle (défaut)** : s'arrête au **bord du snapshot** — la première page lue que le dump n'a pas capturée, ou le premier appel système. Sur un minidump classique (pile + quelques modules), on l'atteint en quelques instructions ; sur un *full dump*, on va beaucoup plus loin.
- **`--keep-going`** : retire cet arrêt en best-effort — pages manquantes mappées à zéro, `syscall`/`int`/`sysenter` sautés, instructions qu'Unicorn ne décode pas sautées. **Dès le premier octet manquant, la trace devient spéculative**, pas la vérité du processus. Signalé en sortie et par `"skipped": true` dans la trace.
- **`--serve`** : expose l'émulateur sur **le protocole du plugin**, si bien que `x64dbg_bridge.Debugger(port=...)` pilote le dump comme une cible vivante (`step_into`, `regs`, `read`, `disasm`...), dans les limites du snapshot.

Depuis x64dbg, `pybridge.emu "crash.dmp"[, port[, thread[, keepgoing]]]` lance `--serve` dans une console. Le script est cherché dans `<release>\pybridge\examples\` (ou via `X64DBG_EMU_SCRIPT`).

## Modèles pour headless.exe

```powershell
.\headless\run-headless.ps1 -Template api-log -Target C:\Windows\System32\whoami.exe -X64dbgDir C:\Tools\x64dbg\release
.\headless\run-headless.ps1 -Template dump-module -Target .\programme.exe -X64dbgDir C:\Tools\x64dbg\release -Arch x32
```

Le lanceur remplace `%TARGET%`, `%MODULE%`, `%OUT%` et `%BITS%` dans le modèle, lance `headless.exe -cf` avec un dossier de configuration isolé (les `settingset` du modèle ne modifient pas votre configuration d'x64dbg) et écrit tout dans `out\<modèle>-<date>\` (`script.txt`, `headless.log`, fichiers produits). Un modèle se termine par `log "[template] done"`.

| Modèle | Rôle |
|---|---|
| `api-log` | journalise les appels d'API avec leurs arguments, sans pause |
| `trace-export` | trace 50 000 instructions depuis le point d'entrée et les exporte en JSON/CSV/Tenet (nécessite `TraceExport` de Fathom) |
| `dump-module` | sauvegarde le module principal (`.mem`) et un minidump complet au point d'entrée |
| `break-on-file` | breakpoint conditionnel (chemin contenant `.txt`) puis actions automatiques : log, sauvegarde de la pile, minidump |
| `antidebug` | lance la cible avec `hide` (PEB patché : `BeingDebugged`, `NtGlobalFlag`, heap flags) et journalise les sondes anti-debug |
| `dump-at-delay` | laisse la cible s'exécuter puis dump le module principal + un minidump (packers qui déballent au démarrage) |
| `hook-unpack` | journalise sans pause les API typiques de dépaquetage runtime (`VirtualAlloc`, `VirtualProtect`, `WriteProcessMemory`, `NtMapViewOfSection`, `CryptDecrypt`, `BCryptDecrypt`) |
| `unpack-dump` | journalise les `VirtualProtect`/`VirtualProtectEx` qui passent en PAGE_EXECUTE* (masque `0xF0`) et dump la **dernière** région (`vprotect-last.bin`) ; pour un dump par appel : `examples/unpack_dump.py` |
| `find-pattern` | `findallmem` sur toute la carte mémoire : motif hexadécimal avec jokers, ou littéral ASCII |
| `run-trace` | enregistre 100 000 instructions depuis le point d'entrée et les exporte en JSON et CSV (nécessite `TraceExport` de Fathom) |
| `hook-apis-extended` | version élargie de `api-log` (contrôle processus/threads, loader, mémoire, I/O, registre, crypto) |
| `emulate-dump` | `-Target` est un `.dmp` : lance `pybridge.emu` et sert l'émulation sur 127.0.0.1:27041 |

Les modèles lisent les arguments avec `arg.get(n)` : ils fonctionnent en x64 et en x32 (`-Arch x32`).

## Anti-anti-debug

Deux niveaux, utilisables ensemble :

- **`dbg.hide_debugger()`** (ou `hide` dans un script) — commande intégrée d'x64dbg. Patche le PEB du processus débogué dès que vous avez la main : `BeingDebugged = 0`, bits de debug (`0x70`) de `NtGlobalFlag` nettoyés, flags de la *process heap* normalisés. Couvre x64 et WoW64. Idempotent ; à appeler après `init()` / `attach()` et avant le premier `run()`. Suffit pour les sondes basiques (`IsDebuggerPresent`, `CheckRemoteDebuggerPresent`, lecture directe du PEB).
- **ScyllaHide** — plugin tiers pour les sondes avancées (`NtQueryInformationProcess`, `NtSetInformationThread`, timing, exceptions...). Déposez ses fichiers (depuis la release ScyllaHide) dans `<x64dbg>\x64\plugins\` et `<x64dbg>\x32\plugins\`, puis choisissez un profil **avant** `init()` / `attach()` :

  ```python
  ini = r"C:\Tools\x64dbg\release\x64\plugins\scylla_hide.ini"
  print(Debugger.scyllahide_profiles(ini))          # ['VMProtect x86/x64', 'Themida x86/x64', ...]
  Debugger.scyllahide_set_profile(ini, "Basic")
  ```

Ces fonctionnalités servent à l'analyse de binaires que vous êtes autorisé à étudier (vos propres programmes, échantillons de malware dans un bac à sable, challenges CTF...). Elles n'affectent que le processus débogué par cette instance d'x64dbg.

## Protocole (pour d'autres langages)

Une requête JSON par ligne, une réponse JSON par ligne :

```
→ {"id": 1, "method": "mem_read", "params": {"addr": "rsp", "size": 16}}
← {"id": 1, "result": "00000000000000009512A857F77F0000"}
← {"id": 2, "error": "cannot read memory"}
```

Méthodes : `ping`, `cmd`, `eval`, `state`, `wait`, `run`, `pause`, `step_into`, `step_over`, `step_out`, `regs`, `reg_get`, `reg_set`, `mem_read`, `mem_write`, `mem_valid`, `disasm` (voir [`src/api.cpp`](src/api.cpp)). Les adresses et registres sont renvoyés en chaînes hexadécimales (`"0x7FF6..."`).

## Compilation

Prérequis : Visual Studio 2022 (charge de travail C++), CMake, Ninja, et le **SDK de plugins** d'x64dbg (le dossier `pluginsdk` d'une [release x64dbg](https://github.com/x64dbg/x64dbg/releases)). Par défaut, le SDK d'un x64dbg installé par winget est utilisé ; sinon, passer `-DX64DBG_SDK=<pluginsdk>`.

```bat
:: 64 bits, après vcvars64.bat
cmake -B build-x64 -G Ninja -DCMAKE_BUILD_TYPE=Release -DX64DBG_SDK=C:\Tools\x64dbg\pluginsdk
cmake --build build-x64

:: 32 bits, après vcvars32.bat
cmake -B build-x32 -G Ninja -DCMAKE_BUILD_TYPE=Release -DX64DBG_SDK=C:\Tools\x64dbg\pluginsdk
cmake --build build-x32
```

## Intégré à

pybridge est livré avec [Fathom](https://github.com/EGKrb/Fathom) (boîte à outils de rétro-ingénierie construite sur x64dbg), qui ajoute `TraceExport` et la visionneuse de dumps. Ce dépôt est sa version autonome. Pour inspecter la mémoire noyau à côté d'une session pybridge, voir [wineye](https://github.com/EGKrb/wineye).

## Licence

GPL-3.0-or-later — voir [LICENSE](LICENSE). pybridge est un plugin pour [x64dbg](https://github.com/x64dbg/x64dbg) (GPL-3.0).
