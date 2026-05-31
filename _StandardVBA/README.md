# elucubrIAmo — sorgente di verità AI-readable VBA

**Stato**: bozza 2, maggio 2026 — **bozza avanti su `Qualità/`**: incorpora la revisione di maggio 2026 (Lead come entità distinta, abolizione dei DoR, Tipologia commessa, Analisi/Kickoff nel gestionale) **prima** della sua formalizzazione nei documenti SQ ufficiali.
**Scopo**: tradurre in formato markdown leggibile da umani e AI il paradigma di gestione commessa VBA, integrando il Sistema Qualità aziendale, il metodo personale dei PM, e l'interfaccia col gestionale **RoadMapp**. È il livello "operativo quotidiano" che istanzia, su filesystem e template concreti, ciò che il SQ codifica in PDF/docx.

---

## 1. Cos'è e cosa non è

**È**:
- Una sintesi machine-readable del paradigma Lead → Commessa VBA (flusso, DoD, RACI, KPI).
- Una libreria di **convenzioni operative** (struttura cartelle, naming, archiviazione) che istanziano la procedura sul filesystem.
- Una libreria di **template markdown** istanziabili nelle cartelle di commessa (Analisi Fattibilità, KickOff, FAT, SAT, CR, Lesson Learned, Rapporto giornaliero).
- Lo "snapshot" leggibile da Claude Code quando lavora dentro la cartella di una commessa.

**Non è**:
- Un sistema qualità alternativo. La sorgente formale resta `G:/Drive condivisi/C - Vba gestione/Qualità/` (PDF/docx ufficiali, certificazione ISO 9001:2015).
- Un manuale RoadMapp. RoadMapp è il gestionale: qui c'è solo l'**interfaccia operativa** (cosa si fa nel gestionale a ogni transizione di stato).
- Un documento finito. È in costruzione e si evolve commessa per commessa.

---

## 2. Mappa dei contenuti

### Procedura master
- [procedura/flusso-commessa.md](procedura/flusso-commessa.md) — flusso Lead → Commessa, conversione (Tipologia + Profilo), stati RoadMapp, transizioni speciali
- [procedura/dod-checklist.md](procedura/dod-checklist.md) — DoD per ogni transizione (DoR aboliti), copia-incolla nelle commesse
- [procedura/ruoli-raci.md](procedura/ruoli-raci.md) — chi fa cosa, derivata da `tabella raci.xlsx`
- [procedura/interfaccia-roadmapp.md](procedura/interfaccia-roadmapp.md) — azioni concrete nel gestionale (sezioni Commesse/Lead/Offerte/Task/Ore) a ogni transizione

### Struttura locale
- [struttura-locale/struttura-lead.md](struttura-locale/struttura-lead.md) — struttura del **Lead** (pre-vendita, drive `B - Lead`)
- [struttura-locale/struttura-commessa.md](struttura-locale/struttura-commessa.md) — convenzioni per `Acquisti/`, `Commerciale/`, `Gestionale/`, `Tecnico/`, `Transit/`, naming file, cosa archivia ogni cartella (drive `A - Commesse`)
- [struttura-locale/profilo-macchina.md](struttura-locale/profilo-macchina.md) — schema del file `~/.vba/profile.json` per disaccoppiare path fisici locali da identità logica (operatore, root commesse, root repo)
- [struttura-locale/gruppo-lavoro.md](struttura-locale/gruppo-lavoro.md) — composizione di un "gruppo di lavoro" (commessa + repo correlati + archivi) e materializzazione in file workspace per IDE (VS Code, Claude Code, Rider, Sublime), portabili per-host

### Discipline tecniche
- [discipline/README.md](discipline/README.md) — overview e flag Progettazione/Manodopera per disciplina
- [discipline/meccanica.md](discipline/meccanica.md)
- [discipline/elettrica.md](discipline/elettrica.md)
- [discipline/pneumatica-idraulica.md](discipline/pneumatica-idraulica.md)
- [discipline/software-hmi.md](discipline/software-hmi.md) — HMI/SCADA non-Siemens (Weintek, Pro-face, ecc.)
- **Software PLC**: vedi [plc/](plc/) — già pilota completo (convenzioni progetto, catalogo UDT, catalogo funzioni base)

### Template istanziabili
- [modelli/analisi-fattibilita.md](modelli/analisi-fattibilita.md) — versione markdown del modello SQ
- [modelli/kickoff.md](modelli/kickoff.md) — versione markdown del verbale kickoff
- [modelli/verbale-fat.md](modelli/verbale-fat.md)
- [modelli/verbale-sat.md](modelli/verbale-sat.md)
- [modelli/change-request.md](modelli/change-request.md)
- [modelli/lesson-learned.md](modelli/lesson-learned.md)
- [modelli/rapporto-giornaliero.md](modelli/rapporto-giornaliero.md) — generalizzato dal pattern PLC

### Specializzazione disciplina software PLC (preesistente)
- [plc/convenzioni-progetto-plc.md](plc/convenzioni-progetto-plc.md)
- [plc/catalogo-udt-base.md](plc/catalogo-udt-base.md)
- [plc/catalogo-funzioni-base.md](plc/catalogo-funzioni-base.md)
- [plc/preanalisi-db-templates.md](plc/preanalisi-db-templates.md)

### Automazioni — skill Claude Code
- [skills/comm-kickoff/](skills/comm-kickoff/) — skill di standardizzazione di una cartella commessa nuova. Copia canonica; per usarla va installata in `~/.claude/skills/`. Vedi [skills/README.md](skills/README.md).

### Memoria di lavoro
- [kickOff/](kickOff/) — appunti seminali, brainstorming
- [rapporti/](rapporti/) — rapporti delle sessioni di standardizzazione

---

## 3. Principio di gerarchia (allineato con `plc/`)

```
Sistema Qualità VBA (PDF/docx in C - Vba gestione/Qualità/)   ← AUTORITÀ FORMALE
    │
    │   (sintesi operativa AI-readable)
    ▼
elucubrIAmo/                                                   ← QUESTO PARADIGMA
    │
    │   (snapshot _StandardVBA/ in ogni commessa)
    ▼
<commessa XXXX>/_StandardVBA/                                  ← copia locale
```

**Regole**:
- Quando una procedura SQ cambia, **prima** si aggiorna `Qualità/`, **poi** si rigenera la sintesi qui in `elucubrIAmo/`, **poi** si propaga lo snapshot nelle commesse attive.
- Se trovi un'incoerenza tra `elucubrIAmo/` e `Qualità/`, prevale `Qualità/`. Apri una nota in `rapporti/` e correggi.
- Le specializzazioni di commessa **non contraddicono** mai `elucubrIAmo/`: o estendono, o si promuovono qui (stesso pattern già usato per `plc/`).

---

## 4. Riferimenti SQ aziendali

Documenti di riferimento (`C - Vba gestione/Qualità/`):

| Documento | Ruolo | Frequenza revisione |
|---|---|---|
| `Flussi di Commessa - VBA.docx` (mar 2026) | **Fonte primaria**: profili workflow, DoD per transizione, discipline. ⚠️ Non ancora aggiornata sulla revisione Lead/Tipologia (mag 2026) | Quando cambiano flusso o DoD |
| `Ciclo commessa.pdf` (mar 2026) | Diagramma stati RoadMapp | Quando cambiano gli stati |
| `Piano organizzativo.pdf` (nov 2025) | Organigramma, RACI, KPI dashboard | Annuale o a evento |
| `tabella raci.xlsx` (nov 2025) | RACI dettagliata (machine-readable) | A evento (nuovo ruolo, nuova attività) |
| `Modelli/Analisi Fattibilità.docx` (mar 2026) | Template ufficiale fase 2 | A revisione |
| `Modelli/KickOff - Nuovo Modulo.docx` (mar 2026) | Template ufficiale fase 4 | A revisione |
| `Procedure/Gestione commessa.pdf` (lug 2025, storico) | Flusso operativo con moduli MV001-MV014 | In via di sostituzione |
| `Procedure/Qualità/iso-9001-2015.pdf` | Riferimento normativo | — |

---

## 5. Convenzioni di scrittura (per chi contribuisce)

- **Markdown puro**, niente HTML inline.
- **Checkbox** con `- [ ]` (non `☐`): più leggibile dall'AI e renderizzabile su GitHub.
- **Link relativi** tra file in `elucubrIAmo/`.
- **Date assolute** in formato ISO (`2026-05-02`), mai relative.
- Ogni file inizia con: titolo, Stato, Scopo, Correlati. Stesso pattern di `plc/convenzioni-progetto-plc.md`.
- Sezioni numerate.
- Tabelle markdown per mappature/RACI/matrici.

---

## 6. Stato attuale e prossimi passi

**Revisione maggio 2026 — separazione Lead/Commessa** (vedi [rapporti/rapporto_20260529.md](rapporti/rapporto_20260529.md)):
- **Lead come entità distinta**: contatti separati dalle commesse. Flusso Lead `Creato → Analisi → Offerta → Attesa ordine → Convertito | Respinto | Annullato`; conversione → Commessa. Drive condivisi separati: `A - Commesse` e **`B - Lead`**.
- **Tipologia commessa** (nuovo asse, scelto alla conversione e ortogonale al profilo): `Preventivo` / `Consuntivo` / `Assistenza generica`.
- **DoR aboliti**: si lavora solo con i DoD. `dor-dod-checklist.md` rinominato `dod-checklist.md`.
- **Analisi Fattibilità compilata nel gestionale** (7 sezioni); **Kickoff** in via di integrazione. I template markdown restano come riferimento/storico.
- **Profili workflow in ridefinizione** lato gestionale: l'elenco e le catene-fasi vanno riallineati (non ancora consolidati).
- Stato di questa doc: **bozza avanti su `Qualità/`** — i documenti SQ ufficiali non sono ancora aggiornati su questa revisione.

**Contesto precedente**:
- Fondamenta scritte (procedura, struttura locale, 4 discipline, 7 template); `plc/` operativo (pilota su 0645 e 669); prima istanza non-PLC **0704 ERRE COMPANY — Banco fatica cofani** (BU Banchi).
- **Migrazione a Google Workspace** (mag 2026): da Microsoft 365/OneDrive a Google Workspace; `elucubrIAmo/` vive nel Drive condiviso `I - iaRoot/`. Vedi [rapporti/rapporto_20260521.md](rapporti/rapporto_20260521.md).

**Da consolidare**:
- **Nomenclatura cartelle `B - Lead`** (prossimo passo dichiarato) — vedi [struttura-locale/struttura-lead.md §4](struttura-locale/struttura-lead.md).
- **Profili workflow**: validare elenco e catene-fasi sul gestionale (incl. il nuovo `Automazione completa`).
- Far "digerire" la nuova procedura al team; formalizzare la revisione in `Qualità/`.
- Validazione 0704 sul campo; skill `/sync-standard-vba`; migrazione moduli storici MV001-MV014 (parziale).

Vedi [rapporti/](rapporti/) per cronologia decisioni.
