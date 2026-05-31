# Profilo macchina locale — convenzioni di accesso ai dati VBA

**Stato**: bozza 1, maggio 2026.
**Scopo**: dichiarare in un file standard, su ogni macchina dove si lavora con dati VBA, **dove sono fisicamente le radici** (commesse, repo, elucubrIAmo) e **chi sta operando**, in modo che workspace, skill e automazioni AI siano portabili senza path assoluti hardcoded.
**Correlati**:
- [struttura-commessa.md](struttura-commessa.md) — usa `<commesse_root>` definito qui
- [../README.md](../README.md) §2 — mappa contenuti elucubrIAmo
- [../procedura/flusso-commessa.md](../procedura/flusso-commessa.md)

---

## 1. Problema che risolve

I dati VBA possono essere acceduti in modi fisicamente diversi a seconda della macchina:

| Modalità di accesso | Esempio root | Tipico per |
|---|---|---|
| Google Drive for Desktop | `G:/Drive condivisi/` (Win), `~/Library/CloudStorage/GoogleDrive-<account>/Drive condivisi/` (Mac) | Tutte le postazioni con accesso ai Drive condivisi VBA |
| Copia locale parziale | path arbitrario | Lavoro offline straordinario |

Entrambe le modalità puntano agli **stessi dati logici**: i dati di rete VBA vivono nei **Drive condivisi** di Google Workspace. Ogni cartella di primo livello di `Drive condivisi/` è un Drive condiviso autonomo (`A - Commesse`, `C - Vba gestione`, `I - iaRoot`, ecc.). Quello che cambia tra macchine è solo il path locale di mount.

Senza disaccoppiare path fisico da identità logica, ogni workspace, skill, automazione AI andrebbe configurata a mano per ogni macchina. Il profilo macchina cancella questo costo.

> **Nota migrazione (maggio 2026)**: VBA è passata da Microsoft 365 a Google Workspace. I path OneDrive (`C:/…/vbanet.eu/VBA Data - Documenti/…`) e l'accesso SMB al NAS `\\VBANAS` sono **dismessi**: tutti i dati di rete sono ora sui Drive condivisi Google. I valori `access_mode` legacy `onedrive_sync` e `smb_mount` restano riconosciuti solo per leggere profili non ancora aggiornati.

---

## 2. Posizione e nome del file

**Path canonico**: `~/.vba/profile.json`

- Windows: `C:/Users/<utente>/.vba/profile.json`
- macOS / Linux: `/Users/<utente>/.vba/profile.json` (o `/home/<utente>/...`)

`~` è espandibile cross-platform da PowerShell, bash, Node, Python. La cartella `.vba/` è una dotfile (invisibile a explorer/finder per default) e su entrambe le piattaforme vive **fuori da qualsiasi cartella sincronizzata in cloud** (Google Drive, iCloud, OneDrive) — esattamente la proprietà desiderata: il profilo è *per-macchina*, non sincronizzato.

**Override**: se serve un profilo alternativo (macchina condivisa, sessione di test), valorizzare la variabile d'ambiente `VBA_PROFILE` con un path assoluto. Il consumer del profilo (skill, script) deve leggere prima `$VBA_PROFILE` e fallback su `~/.vba/profile.json`.

---

## 3. Schema (`$schema_version: 1`)

```json
{
  "$schema_version": 1,
  "operator": {
    "id": "roberto",
    "name": "Roberto Visca",
    "email": "stormdev87@gmail.com"
  },
  "host": "rovi-win-fissa",
  "access_mode": "gdrive_sync",
  "roots": {
    "vba_data":    "G:/Drive condivisi",
    "commesse":    "G:/Drive condivisi/A - Commesse",
    "lead":        "G:/Drive condivisi/B - Lead",
    "elucubriamo": "G:/Drive condivisi/I - iaRoot/elucubrIAmo",
    "github":      "C:/Documenti/GitHub"
  }
}
```

### 3.1 Campi

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `$schema_version` | int | sì | Per migrazioni future. Versione corrente: `1`. |
| `operator.id` | slug | sì | Identificativo breve lowercase (`roberto`, `matteo`, `andrea`, `paolo`, `mattso`). Usato in rapporti, log, attribuzioni. |
| `operator.name` | string | sì | Nome completo human-readable. |
| `operator.email` | string | no | Mail VBA o personale lavorativa. |
| `host` | slug | sì | Identifica la **macchina** (es. `rovi-win-fissa`, `rovi-win-portatile`, `andrea-macmini`). Necessario perché lo stesso operatore può lavorare da più macchine con `roots` diverse. |
| `access_mode` | enum | sì | `gdrive_sync` \| `local_copy`. Valori legacy pre-migrazione (`onedrive_sync`, `smb_mount`) ancora accettati in lettura. Informativo nella v1; può pilotare comportamenti futuri (es. segnalare lavoro offline nei rapporti). |
| `roots.vba_data` | path assoluto | sì | Radice dei Drive condivisi Google — il livello sopra i singoli Drive (`A - Commesse`, `C - Vba gestione`, `I - iaRoot`, ecc.). Su Windows tipicamente `G:/Drive condivisi`. |
| `roots.commesse` | path assoluto | sì | Radice che contiene le cartelle commessa. Di norma `<vba_data>/A - Commesse`, ma dichiarato esplicito (alcune macchine potrebbero mappare solo questa sottocartella). |
| `roots.lead` | path assoluto | no | Radice del drive Lead (pre-vendita). Di norma `<vba_data>/B - Lead`. Introdotto con la separazione Lead/Commessa (mag 2026); opzionale per retrocompatibilità con profili più vecchi — vedi [struttura-lead.md](struttura-lead.md). |
| `roots.elucubriamo` | path assoluto | sì | Radice di elucubrIAmo (sorgente di verità doc). Di norma `<vba_data>/I - iaRoot/elucubrIAmo`. |
| `roots.github` | path assoluto | sì | Radice locale dei repo git VBA (es. `C:/Documenti/GitHub`, `/Users/<x>/dev/vba`). |

### 3.2 Convenzioni path

- **Forward slash** `/` anche su Windows: JSON-friendly, gestito da tutti i runtime.
- **Niente trailing slash**.
- Path **assoluti**, mai relativi.
- Niente espansione `~` dentro i `value`: scrivere il path completo. (`~` espansa solo per `VBA_PROFILE`, §2.)

### 3.3 Niente segreti

Il profilo è **pointer + identità**, non credenziali. Mai metterci token, password, chiavi API. Le credenziali vivono altrove (Windows Credential Manager, macOS Keychain, gestori dedicati).

---

## 4. Esempi

### 4.1 Roberto — Windows con Google Drive for Desktop (caso corrente)

```json
{
  "$schema_version": 1,
  "operator": { "id": "roberto", "name": "Roberto Visca", "email": "stormdev87@gmail.com" },
  "host": "rovi-win-fissa",
  "access_mode": "gdrive_sync",
  "roots": {
    "vba_data":    "G:/Drive condivisi",
    "commesse":    "G:/Drive condivisi/A - Commesse",
    "lead":        "G:/Drive condivisi/B - Lead",
    "elucubriamo": "G:/Drive condivisi/I - iaRoot/elucubrIAmo",
    "github":      "C:/Documenti/GitHub"
  }
}
```

### 4.2 Postazione officina — Windows con Google Drive for Desktop

```json
{
  "$schema_version": 1,
  "operator": { "id": "matteo", "name": "Matteo Visca" },
  "host": "officina-win-banco3",
  "access_mode": "gdrive_sync",
  "roots": {
    "vba_data":    "G:/Drive condivisi",
    "commesse":    "G:/Drive condivisi/A - Commesse",
    "lead":        "G:/Drive condivisi/B - Lead",
    "elucubriamo": "G:/Drive condivisi/I - iaRoot/elucubrIAmo",
    "github":      "C:/dev/vba"
  }
}
```

### 4.3 Andrea — Mac con Google Drive for Desktop

```json
{
  "$schema_version": 1,
  "operator": { "id": "andrea", "name": "Andrea Visca" },
  "host": "andrea-macmini",
  "access_mode": "gdrive_sync",
  "roots": {
    "vba_data":    "/Users/andrea/Library/CloudStorage/GoogleDrive-andrea@<dominio>/Drive condivisi",
    "commesse":    "/Users/andrea/Library/CloudStorage/GoogleDrive-andrea@<dominio>/Drive condivisi/A - Commesse",
    "lead":        "/Users/andrea/Library/CloudStorage/GoogleDrive-andrea@<dominio>/Drive condivisi/B - Lead",
    "elucubriamo": "/Users/andrea/Library/CloudStorage/GoogleDrive-andrea@<dominio>/Drive condivisi/I - iaRoot/elucubrIAmo",
    "github":      "/Users/andrea/dev/vba"
  }
}
```

> *Nota Mac*: il path Google Drive for Desktop su macOS (`~/Library/CloudStorage/GoogleDrive-<account>/`) dipende dall'account Google Workspace e dalla versione del client — verificare sulla postazione reale. L'esempio §4.3 è una stima da confermare.

---

## 5. Bootstrap di una macchina nuova

### 5.1 Modalità raccomandata: bootstrap guidato dall'AI

Il modo più semplice è **lasciare che Claude crei il file in interview**:

1. Garantire l'accesso ai Drive condivisi VBA via Google Drive for Desktop (account Google Workspace VBA abilitato, Drive condivisi montati localmente).
2. Aprire una sessione Claude con `cwd` dentro `elucubrIAmo/` (oppure citare elucubrIAmo nel primo prompt).
3. Claude rileva l'assenza di `~/.vba/profile.json`, deriva i path derivabili dal cwd e dall'OS, chiede all'utente solo i campi mancanti (operator, host, access_mode, github root), verifica leggibilità, mostra il JSON e scrive il file dopo OK esplicito.
4. Protocollo dettagliato in [../CLAUDE.md §3](../CLAUDE.md).

### 5.2 Modalità manuale

Se si lavora senza assistente AI, o si preferisce comunque fare a mano:

1. Creare la cartella `~/.vba/` (Win: `C:/Users/<utente>/.vba/`; Mac/Linux: `~/.vba/`).
2. Copiarvi `profile.json` partendo dall'esempio §4 più vicino al setup della macchina e adattando i path.
3. Verificare che ognuno dei 4 `roots` sia leggibile (`ls` / `Get-ChildItem` deve rispondere).
4. Da questo momento ogni skill/workspace/Claude session basata sul profilo macchina lavora portabile.

---

## 6. Come Claude / le skill leggono il profilo

- **Skill kickoff locale**: legge `profile.json` come prima cosa, riceve `<NNNN>` come argomento, risolve la cartella commessa come `<roots.commesse>/<NNNN - CLIENTE - …>/`, eventuali repo gemelli come `<roots.github>/<NNNN>_<mnemonico>_plc/` e `<roots.github>/<NNNN>_<mnemonico>/`, e genera i file workspace per gli IDE pertinenti seguendo [gruppo-lavoro.md](gruppo-lavoro.md) (per VS Code: `<NNNN>.<host>.code-workspace` nella root della commessa).
- **Generazione rapporti**: usa `operator.id` come campo "autore", `host` come campo "postazione". Vedi [../modelli/rapporto-giornaliero.md](../modelli/rapporto-giornaliero.md).
- **Cross-link commessa ↔ repo**: la skill costruisce il **gruppo di lavoro** (vedi [gruppo-lavoro.md](gruppo-lavoro.md)) combinando `roots.commesse` (cartella commessa) e `roots.github` (eventuali repo). Se una commessa non ha repo (caso comune: lead non-software), il gruppo di lavoro contiene solo la cartella commessa.
- **Regola hard**: nessuno script/skill può contenere path assoluti VBA hardcoded nel sorgente. Se serve un path, va risolto via `profile.json`. Una violazione è un bug.

---

## 7. Stato e prossimi passi

**Maggio 2026** — versione iniziale `$schema_version: 1`. Schema esplicito, niente derivazioni implicite (tutti i `roots` dichiarati a mano per chiarezza). Esempi e modello allineati a Google Workspace dopo la migrazione di maggio 2026 (vedi [../rapporti/rapporto_20260521.md](../rapporti/rapporto_20260521.md)).

**Da consolidare**:
- Skill kickoff locale: **creata** come `comm-kickoff` (legge il profilo, standardizza la cartella commessa, genera workspace file VS Code per-host secondo [gruppo-lavoro.md](gruppo-lavoro.md) §4.1 — vedi `../skills/`). Estensione futura: skill `/workspace-build` per rigenerare solo i workspace file senza ripetere il kickoff completo.
- Validazione su almeno due macchine fisiche diverse (Roberto Win + una seconda) prima di propagare a tutto il team.
- Conferma del path Mac `~/Library/CloudStorage/GoogleDrive-<account>/Drive condivisi/` su una postazione Mac reale (esempio §4.3 oggi è una stima).
- Possibile evoluzione `$schema_version: 2` con derivazioni implicite (`commesse` derivato da `vba_data` se assente) dopo consolidamento.
- Eventuale aggiunta di campi (`role`, `default_workflow_profile`, `lang`, `tz`) se l'uso ne mostra il bisogno.
