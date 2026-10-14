# AVR Firmware Template

Template per progetti **Arduino AVR / Visual Micro**: tracciabilità del firmware, compilazione automatica su GitHub e controllo delle release.

## Cosa fa

| Meccanismo | Dove viene eseguito | Risultato |
| --- | --- | --- |
| **Versionamento firmware** | **PC Windows, Visual Micro** | Genera `GitVersion.h` prima di ogni compilazione |
| **Firmware CI** (`firmware-ci.yml`) | **GitHub Actions** | Compila/verifica il progetto AVR su push e PR |
| **Branch Policy** (`branch-policy.yml`) | **GitHub Actions** | Controlla che le PR verso `main` provengano solo da `release/*` |

## Versione inserita nel firmware

| Compilazione | `GIT_VERSION` |
| --- | --- |
| **Locale** da `develop`, `feature/*`, `main` o altre branch non release | `"XXXXXXX"` |
| **Locale** da `release/*` valida | Hash di **7 caratteri** del commit Git (es. `"a1b2c3d"`) |
| **CI remota** su GitHub (anche per `release/*`) | `"CI"` |

Lo script `GenerateGitVersion.bat`, richiamato automaticamente da `board.txt`, genera `GitVersion.h` **già alla prima compilazione** e lo rigenera a ogni build. Per usare la macro nel codice: `#include "GitVersion.h"` (con percorso di inclusione corretto).

**Una release locale è valida soltanto se** la working tree Git è pulita e il commit locale coincide con quello della branch `origin/release/*`. In caso contrario **la compilazione fallisce**. L'hash incorporato permette di identificare il commit da cui deriva il firmware.

## Dove vanno i file

```text
progetto/
├── board.txt                 # pre-build Visual Micro
├── GenerateGitVersion.bat    # genera GitVersion.h
├── .gitignore                # esclude i file generati
└── .github/workflows/
    ├── branch-policy.yml    # verifica PR verso main
    └── firmware-ci.yml      # compilazione AVR su GitHub
```

`board.txt` e `GenerateGitVersion.bat` devono stare **nella root del progetto Visual Micro** (`{build.project_path}`); può coincidere con la root del repository. `GitVersion.h` e `GitVersion.tmp` **non vanno committati**.

## Prima di usare il template

1. Creare il repository con **Use this template** e aggiungere il firmware.
2. Adattare in `firmware-ci.yml` il percorso `FIRMWARE_PATH`, il core `ARDUINO_CORE`, la scheda `FQBN` e le **librerie** necessarie (step `Prepare firmware libraries`). Per impostazione iniziale: Arduino Nano AVR e librerie da `MicroControllerTeamProject/libraries`, branch `develope`.
3. Adottare il flusso **`develop → release/* → main`** e impostare su GitHub la protezione di `main`, rendendo obbligatorio il check **`Validate source branch`**: il workflow, da solo, **non impedisce il merge**.
4. Per la generazione **locale** dell'hash servono Git, Windows e Visual Micro con il pre-build attivo.

**Nota:** la CI remota verifica la compilazione, ma **non programma il dispositivo** e **non genera l'hash reale della release**.
