# Gruppo di lavoro — composizione e file workspace per IDE

**Stato**: bozza 1, maggio 2026.
**Scopo**: definire cos'e' un "gruppo di lavoro" attorno a una commessa (cartella commessa Drive + repo correlati + eventuali archivi) e come si materializza in file di workspace per gli IDE usati dal team. L'obiettivo: aprire una commessa col contesto completo in due click, senza ricostruire path a memoria, in modo portabile fra postazioni.
**Correlati**:
- [profilo-macchina.md](profilo-macchina.md) — i path locali per-host
- [struttura-commessa.md](struttura-commessa.md) §7.3 — `LINKS.md` elenca i repo correlati
- [../skills/comm-kickoff/SKILL.md](../skills/comm-kickoff/SKILL.md) — genera i workspace file al kickoff

---

## 1. Cos'e' un "gruppo di lavoro"

Una commessa VBA tipicamente coinvolge piu' filesystem root distinte:
- La **cartella commessa** sul Drive condiviso (`<roots.commesse>/<NNNN - CLIENTE - …>/`)
- Uno o piu' **repo git** correlati (PLC, BE/FE, HMI esterno)
- Eventuali **archivi** locali (es. backup `.zap` PLC in `Vph-PLCz/<NNNN>_<mnemonico>/`)
- Eventuali **risorse esterne** referenziate (cartelle vendor, doc fornitore, libreria componenti)

Lavorare sulla commessa significa avere queste root **simultaneamente visibili nell'IDE**, non aprire N finestre separate per cercare il file giusto.

Il **gruppo di lavoro** e' l'insieme delle root pertinenti a una commessa. La sua composizione evolve nel tempo: una commessa puo' partire con la sola cartella Drive (lead non-software), aggiungere un repo PLC al kick-off, aggiungere un repo BE/FE quando parte lo sviluppo software, eventualmente aggiungere un repo HMI esterno o un archivio post-chiusura.

---

## 2. Composizione tipica per profilo

| Profilo / fase | Root tipiche del gruppo di lavoro |
|---|---|
| Lead / commessa non-software | Solo cartella commessa Drive |
| Commessa con PLC | Commessa Drive + repo PLC |
| Commessa con PLC + software gestionale | Commessa Drive + repo PLC + repo BE/FE |
| Commessa con HMI esterno | + repo HMI |
| Post-chiusura con archivio | + cartella `<roots.vba_data>/G - Archivio/Vph-PLCz/<NNNN>_<mnemonico>/` (se esiste) |
| Riferimento a libreria componenti | + cartella vendor / libreria standard |

Le root vivono in **volumi e account diversi**:
- Cartella commessa, archivi → `<roots.vba_data>` / `<roots.commesse>` (Drive condiviso Google)
- Repo git → `<roots.github>` (locale, per-macchina)

→ I path sono **macchina-specifici**. La portabilita' si ottiene **risolvendo via [profilo macchina](profilo-macchina.md)** (`~/.vba/profile.json`), mai con path assoluti hardcoded.

Su una macchina priva di profilo, eseguire prima il **bootstrap interview** del profilo descritto in [../CLAUDE.md §3](../CLAUDE.md) e [profilo-macchina.md §5](profilo-macchina.md). Senza profilo, nessun workspace puo' essere generato in modo portabile.

---

## 3. Portabilita' del workspace file

I file workspace IDE-specifici (es. VS Code `.code-workspace`) contengono **path assoluti** del filesystem. Vivono sul Drive condiviso (sincronizzati fra postazioni) ma i path locali differiscono fra Win/Mac/postazioni con `roots.github` diverso.

**Pattern raccomandato: workspace file per-host.** Nome: `<NNNN>.<host>.code-workspace` (e analoghi per altri IDE). Vantaggi:
- Piu' postazioni coesistono sul Drive senza conflitti
- Il nome e' autodescrittivo (ogni operatore sa qual e' "il suo")
- La skill `comm-kickoff` genera quello della postazione corrente leggendo `host` e i `roots.*` dal profilo

Esempio: una commessa lavorata da Roberto (`asusvrb`) e da una postazione officina (`officina-banco3`) avra' nella root della cartella commessa due file: `<NNNN>.asusvrb.code-workspace` e `<NNNN>.officina-banco3.code-workspace`.

**Alternative scartate**:
- *Single file, last-writer wins* (`<NNNN>.code-workspace`): rompe agli altri quando un host con layout diverso lo rigenera. OK solo se **tutte** le postazioni hanno layout identico (es. tutte Windows con `roots.github = C:/Documenti/GitHub` per convenzione di team). In quel caso si puo' optare per il single-file.
- *Single file con path relativi*: VS Code supporta `path` relativi al file workspace, ma commessa Drive e repo GitHub stanno tipicamente su volumi diversi → non risolvibile con un relative path comune.

---

## 4. Materializzazione per IDE

### 4.1 VS Code (e cloni: Cursor, Windsurf, VSCodium)

**File**: `<NNNN>.<host>.code-workspace` nella **root della cartella commessa**. Formato JSON.

```json
{
  "folders": [
    { "name": "<NNNN> - Commessa (Drive)",  "path": "." },
    { "name": "<NNNN> - PLC (repo)",        "path": "<roots.github>/<NNNN>_<mnemonico>_plc" },
    { "name": "<NNNN> - BE/FE (repo)",      "path": "<roots.github>/<NNNN>_<mnemonico>" }
  ],
  "settings": {
    "files.exclude": { "**/.git": true }
  }
}
```

- `path = "."` e' relativo al file workspace → portabile fra macchine per la sola root commessa.
- Gli altri `path` sono assoluti, risolti dal profilo al momento della generazione.
- Convenzione `name`: prefisso `<NNNN>` + ruolo (`Commessa (Drive)`, `PLC (repo)`, `BE/FE (repo)`, `HMI (repo)`, `Archivio`).

**Apertura**: `File → Open Workspace from File...` selezionando il file con `<host>` corrispondente.

### 4.2 Claude Code

Claude Code non ha un "workspace file"; ha **una root primaria (cwd) + additional directories**, configurabili in `.claude/settings.json` o `.claude/settings.local.json` (per-progetto), oppure via flag CLI `--add-dir`.

**Pattern**:
- Si sceglie una **root primaria** in base a dove parte tipicamente la sessione (di solito il repo PLC se la sessione parte dall'IDE su quel repo; oppure la cartella commessa se si lavora prevalentemente sui documenti).
- Si dichiarano le altre root del gruppo di lavoro in `permissions.additionalDirectories` della root primaria.

Esempio in `<repo PLC>/.claude/settings.local.json`:
```json
{
  "permissions": {
    "additionalDirectories": [
      "<roots.commesse>/<NNNN - CLIENTE - …>",
      "<roots.github>/<NNNN>_<mnemonico>"
    ]
  }
}
```

In alternativa, avvio da CLI:
```bash
claude --add-dir "<roots.commesse>/<NNNN - CLIENTE - …>" \
       --add-dir "<roots.github>/<NNNN>_<mnemonico>"
```

`settings.local.json` non e' versionato in git (e' nel `.gitignore` standard di Claude Code) → e' di fatto per-macchina, coerente col paradigma.

### 4.3 JetBrains Rider / Visual Studio (BE/FE .NET)

Niente workspace multi-root nativo: questi IDE aprono un singolo `.sln` o progetto. Si apre il `.sln` del repo BE/FE come progetto singolo. Per il riferimento alla cartella commessa e al repo PLC si usano:
- Segnalibri / "Recent Locations"
- Finestra "External Tools" per launch verso path noti
- Apertura parallela in un altro IDE (VS Code col workspace multi-root) per i file non-.NET

In `LINKS.md` della commessa: documentare il path del `.sln` come pointer ufficiale.

### 4.4 Sublime Text

**File**: `<NNNN>.<host>.sublime-project` nella root della cartella commessa. Schema simile a VS Code:
```json
{
  "folders": [
    { "name": "<NNNN> - Commessa (Drive)", "path": "." },
    { "name": "<NNNN> - PLC (repo)",       "path": "<roots.github>/<NNNN>_<mnemonico>_plc" }
  ]
}
```

### 4.5 Vim / Emacs / shell

Niente workspace nativo. Si naviga via `cd`, alias shell, `fzf`/`tmux`, plugin tipo `projectile` (Emacs). In `LINKS.md` elencare i path principali, eventualmente preparare un piccolo script `open-0NNN.sh` / `.ps1` per-host che lancia gli editor con le root giuste.

---

## 5. Dove vivono i workspace file

**Root della cartella commessa**, accanto a `README.md` / `CLAUDE.md` / `LINKS.md`. Coerente col fatto che la commessa e' il "punto di ingresso semantico" del lavoro.

Naming canonico:

| IDE | Pattern nome file |
|---|---|
| VS Code (e cloni) | `<NNNN>.<host>.code-workspace` |
| Sublime Text | `<NNNN>.<host>.sublime-project` |
| Claude Code | nessun file dedicato (vedi §4.2) |
| Rider / Visual Studio | nessun file dedicato (vedi §4.3); path `.sln` in `LINKS.md` |

Per la prima postazione, se la commessa e' nuova e nessuno ha ancora generato un workspace, il nome puo' essere ambiguo. Convenzione: lasciar fare alla skill `comm-kickoff`, che usa `host` dal profilo.

`LINKS.md` della commessa (vedi [struttura-commessa.md §7.3](struttura-commessa.md)) deve avere una sezione che elenca i workspace file disponibili e per chi sono.

---

## 6. Quando aggiornare

Il workspace file (per ogni host) va aggiornato quando:
- Si **aggiunge** una root al gruppo di lavoro (es. nasce il repo BE/FE dopo che esisteva solo il PLC).
- Si **rimuove** una root (es. abbandono di un repo).
- Cambia `<roots.github>` o `<roots.commesse>` nel profilo macchina di quell'host (rigenerazione).
- Si lavora da una **postazione nuova**: si genera il file per quel `<host>`.

**Aggiornamento manuale**: aprire `<NNNN>.<host>.code-workspace`, aggiungere/rimuovere voci in `folders[]`, salvare. Annotare in `LINKS.md` se la composizione del gruppo di lavoro e' cambiata (e' un evento strutturale).

**Aggiornamento automatico**: la skill `comm-kickoff` rigenera per la postazione corrente al primo uso e a ogni invocazione successiva. Per una rigenerazione "solo workspace" (commessa gia' standardizzata, ad esempio quando si vuole aggiungere un repo nuovo), in futuro: skill `/workspace-build` o flag `--workspace-only` di `comm-kickoff`. Fino ad allora, modifica manuale.

---

## 7. Cross-link operativi

- Definizione di `<host>` e dei `roots.*` → [profilo-macchina.md](profilo-macchina.md).
- Bootstrap del profilo su macchina nuova → [../CLAUDE.md §3](../CLAUDE.md) e [profilo-macchina.md §5](profilo-macchina.md).
- `LINKS.md` della commessa (sezione workspace) → [struttura-commessa.md §7.3](struttura-commessa.md).
- Generazione automatica al kickoff → [../skills/comm-kickoff/SKILL.md](../skills/comm-kickoff/SKILL.md) §5b (workspace file).
