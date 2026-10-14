# AVR Firmware Template

Template per avviare progetti firmware **AVR (Arduino / Visual Micro)** con versionamento Git, verifica delle release, compilazione automatica e regole per le Pull Request.

## 1. Creare un progetto dal template

1. Su GitHub scegliere **Use this template** → **Create a new repository**.
2. Inserire il codice del firmware nel nuovo repository.
3. Lasciare i workflow in `.github/workflows/`; GitHub Actions li riconosce automaticamente.
4. Per la compilazione **locale su Windows con Visual Micro**, tenere **`board.txt`** e **`GenerateGitVersion.bat`** insieme nella **cartella radice del progetto Visual Micro** (quella identificata da `{build.project_path}`). Se la root del progetto Visual Micro e quella del repository Git coincidono, i file stanno nella root del repository.
5. Adattare `firmware-ci.yml` alla scheda, al percorso del codice e alle librerie usate. Il template **non contiene già un firmware da compilare**.

**Prerequisiti locali:** Git disponibile da riga di comando; Windows con Visual Micro configurato per utilizzare il pre-build definito in `board.txt`. Lo script `.bat` non viene eseguito direttamente da Linux/macOS o da GitHub Actions.

## 2. Versione firmware automatica

Durante la **prima compilazione locale con Visual Micro**, `board.txt` esegue `GenerateGitVersion.bat` prima del compilatore. Lo script crea `GitVersion.h` e lo **rigenera a ogni build**.

Nel firmware, `GitVersion.h` rende disponibile la macro `GIT_VERSION` (ad esempio con `#include "GitVersion.h"`, se il percorso di inclusione è configurato correttamente).

| Branch della compilazione locale | `GIT_VERSION` |
| --- | --- |
| `develop`, `feature/*`, `main` e tutte le branch diverse da `release/*` | `"XXXXXXX"` |
| `release/*` valida | **Hash Git abbreviato a 7 caratteri** del commit, es. `"a1b2c3d"` |

L'hash reale **non viene assegnato in fase di commit**, ma al momento della compilazione locale della release. Consente di risalire al commit corrispondente su GitHub.

### Controlli obbligatori per `release/*`

Prima di generare l'hash, lo script verifica che:

- non vi siano modifiche locali non committate o file non tracciati (working tree pulita);
- sia possibile recuperare la branch remota da `origin`;
- il commit locale `HEAD` coincida con il commit remoto recuperato.

Se uno dei controlli fallisce, viene generato un `GitVersion.h` contenente un `#error`, così **la compilazione della release fallisce**. Perciò una release valida richiede il commit e il push delle modifiche, oltre alla sincronizzazione con GitHub e alla disponibilità della rete.

`GitVersion.h` e `GitVersion.tmp` sono **file generati**: `.gitignore` li esclude dai commit. Non vanno creati né modificati manualmente.

**Attenzione alla differenza fra compilazione locale e CI:** il workflow `firmware-ci.yml` genera attualmente `GIT_VERSION = "CI"` anche quando compila una branch `release/*`. L'hash reale con i controlli sopra descritti è una funzione dello **script locale Visual Micro**, non della CI GitHub.

## 3. Branch e regole di rilascio

Flusso previsto: `feature/*` → `develop` → `release/*` → `main`.

Il workflow **`.github/workflows/branch-policy.yml`** verifica le Pull Request indirizzate a `main`:

- **Accetta** come sorgente soltanto `release/*`.
- **Rifiuta** PR provenienti direttamente da `develop`, `feature/*` o altre branch.

Per rendere questa verifica un **blocco effettivo del merge**, nel repository creato dal template bisogna configurare una regola di protezione di `main` (branch protection o ruleset) che renda obbligatorio il controllo **`Validate source branch`**. Il solo workflow segnala il fallimento, ma non configura automaticamente la protezione GitHub. Occorre proteggere `main` anche dai push diretti, se questa è la policy desiderata.

## 4. Firmware CI (GitHub Actions)

Il workflow **`.github/workflows/firmware-ci.yml`** compila il firmware con **Arduino CLI** su un runner Ubuntu:

- automaticamente su **push** e **Pull Request** che interessano `develop`, `main` o `release/**`;
- manualmente tramite **Run workflow** in GitHub Actions.

Parametri iniziali da controllare **per ciascun nuovo progetto**:

| Parametro | Valore iniziale | Significato |
| --- | --- | --- |
| `ARDUINO_CORE` | `arduino:avr` | Core da installare |
| `FQBN` | `arduino:avr:nano` | Scheda da compilare |
| `FIRMWARE_PATH` | `firmware` | Cartella dello sketch/firmware |
| `SHARED_LIBRARIES_REPOSITORY` | `MicroControllerTeamProject/libraries` | Librerie condivise |
| `SHARED_LIBRARIES_REF` | `develope` | Branch delle librerie condivise |

**Da completare nel progetto:** la cartella indicata da `FIRMWARE_PATH` deve esistere e lo step **Prepare firmware libraries** va personalizzato per copiare in `ci-libraries` le librerie effettivamente necessarie. La branch delle librerie è chiamata proprio `develope` nel repository condiviso.

La CI esegue **solo la compilazione/verifica**, non programma automaticamente il microcontrollore e, nella configurazione attuale, non pubblica un binario di release.

## 5. File inclusi e posizione

| File | Posizione / funzione |
| --- | --- |
| `board.txt` | Root del progetto Visual Micro; attiva il pre-build |
| `GenerateGitVersion.bat` | Stessa cartella di `board.txt`; genera e valida `GitVersion.h` |
| `.github/workflows/branch-policy.yml` | Workflow GitHub per le PR verso `main` |
| `.github/workflows/firmware-ci.yml` | Workflow GitHub di compilazione AVR |
| `.gitignore` | Esclude `GitVersion.h` e `GitVersion.tmp` |

**In sintesi:** nelle build locali di sviluppo il firmware contiene `XXXXXXX`; nelle build locali `release/*` valide contiene l'hash del commit pubblicato; nelle build GitHub Actions contiene `CI`. Le policy di merge devono essere rese obbligatorie nelle impostazioni del repository di destinazione.
