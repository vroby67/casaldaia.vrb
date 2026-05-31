# Struttura locale di commessa — convenzioni filesystem

**Stato**: bozza 2, maggio 2026 — aggiornata alla separazione Lead/Commessa (drive `A - Commesse` vs `B - Lead`).
**Scopo**: definire la struttura di cartelle e la convenzione di archiviazione file per ogni commessa, in modo che (a) un essere umano si orienti in 30 secondi, (b) Claude/AI trovi il contesto giusto entrando nella root, (c) le operazioni del PM (vedi [../procedura/interfaccia-roadmapp.md](../procedura/interfaccia-roadmapp.md)) abbiano un riscontro fisico univoco.

> **Ambito (maggio 2026)**: questo documento copre la **Commessa** (post-conversione, drive `A - Commesse`). La fase pre-vendita vive ora come **Lead** nel drive `B - Lead` — vedi [struttura-lead.md](struttura-lead.md).

**Correlati**:
- [../procedura/flusso-commessa.md](../procedura/flusso-commessa.md)
- [../procedura/dod-checklist.md](../procedura/dod-checklist.md)
- [struttura-lead.md](struttura-lead.md) — struttura del Lead (pre-vendita, drive `B - Lead`)
- [../discipline/](../discipline/) — convenzioni per disciplina dentro `Tecnico/Progetto/`
- [../plc/convenzioni-progetto-plc.md §2](../plc/convenzioni-progetto-plc.md) — pattern già consolidato per l'eventuale repo `_plc` correlato

---

## 1. Posizione e nome cartella commessa

**Posizione**: `<commesse_root>/<NNNN - CLIENTE - Descrizione>/`, dove `<commesse_root>` è dichiarato nel [profilo macchina locale](profilo-macchina.md) (`roots.commesse`). Esempi tipici di `<commesse_root>`:
- Windows con Google Drive for Desktop: `G:/Drive condivisi/A - Commesse`
- Mac con Google Drive for Desktop: `~/Library/CloudStorage/GoogleDrive-<account>/Drive condivisi/A - Commesse` (path tipico, da confermare sulla postazione)

**Nome cartella**: `NNNN - CLIENTE - Descrizione`
- `NNNN` = numero commessa progressivo (4 cifre).
- `CLIENTE` = ragione sociale sintetica del cliente in MAIUSCOLO.
- `Descrizione` = nome breve del progetto in CamelCase con spazi (es. `Banco fatica cofani`).
- Separatori: ` - ` (spazio-trattino-spazio).

**Esempi**:
- `0645 - VITESCO - Banchi pompe trasm`
- `0704 - ERRE COMPANY - Banco fatica cofani`
- `669 - MULLER - Termoregolazione olio pressa`

**Cartella sincronizzata**: vive in un Drive condiviso Google, accessibile da tutte le postazioni via Google Drive for Desktop.

---

## 2. Struttura standard di primo livello

```
<NNNN - CLIENTE - Descrizione>/
├── Acquisti/
│   ├── DDT/
│   ├── Offerte/
│   └── Ordini/
├── Commerciale/
│   ├── Email Importanti/
│   ├── Ordine Cliente/
│   └── Preventivo Fattureincloud/
├── Gestionale/
├── Tecnico/
│   ├── Documentazione cliente/
│   ├── Fascicolo Tecnico/
│   │   ├── Certificazione/
│   │   ├── Manuale/
│   │   └── Schede Tecniche/
│   ├── Foto/
│   ├── Materiali/
│   │   ├── a. meccanico/
│   │   ├── b. fluidico/
│   │   ├── c. elettrico/
│   │   └── d. software/
│   └── Progetto/
│       ├── a. meccanico/
│       │   ├── dwg/
│       │   └── swk/
│       ├── b. fluidico/
│       ├── c. elettrico/
│       └── d. software/
├── Transit/
│
├── Gestione commessa/                    ← NUOVA: tracciamento operativo (vedi §4)
│   ├── _todo.md
│   ├── dod-checklist.md
│   ├── kickoff.md
│   ├── analisi-fattibilita.md
│   ├── rapporti/                          ← rapporti giornalieri
│   ├── meeting/                           ← verbali riunioni interne
│   └── change-requests/                   ← CR firmate
│
├── Contesto/                             ← NUOVA: input read-only di progetto (vedi §5)
│   ├── descrizione-progetto.md
│   ├── capitolato/
│   ├── schemi/
│   │   ├── elettrici/
│   │   ├── fluidici/
│   │   └── meccanici/
│   ├── datasheet/
│   │   ├── plc/
│   │   ├── hmi/
│   │   ├── sensori/
│   │   ├── attuatori/
│   │   └── drive/
│   └── manuali/
│
├── InterazioniCliente/                   ← NUOVA: comunicazioni vive con cliente
│   ├── email/
│   ├── richieste/
│   ├── verbali-cliente/
│   └── CR/                                ← Change Request firmate cliente
│
├── _StandardVBA/                         ← NUOVA: snapshot doc standard (vedi §6)
│   ├── _VERSIONE.md
│   ├── README.md
│   ├── procedura/
│   ├── struttura-locale/
│   ├── discipline/
│   ├── modelli/
│   └── plc/                               ← se applicabile (disciplina PLC)
│
├── README.md                             ← NUOVA: scheda commessa AI/human-readable
├── CLAUDE.md                             ← NUOVA: istruzioni AI per la commessa (opzionale)
└── LINKS.md                              ← NUOVA: legami esterni (RoadMapp, repo git, ecc.)
```

**Cartelle storiche obbligate** (non rinominare): `Acquisti/`, `Commerciale/`, `Gestionale/`, `Tecnico/`, `Transit/`. Sono il telaio condiviso da tutta VBA.

**Cartelle nuove introdotte da `elucubrIAmo/`**: `Gestione commessa/`, `Contesto/`, `InterazioniCliente/`, `_StandardVBA/`, più i tre file root (`README.md`, `CLAUDE.md`, `LINKS.md`).

---

## 3. Cartelle storiche — chi popola cosa

| Cartella | Natura | Cosa contiene | Chi popola |
|---|---|---|---|
| `Acquisti/Offerte/` | Input | Offerte di fornitori per materiali della commessa | PMC + Acquisti (Valentina) |
| `Acquisti/Ordini/` | Output | Nostri ordini emessi a fornitori | Acquisti (Valentina) |
| `Acquisti/DDT/` | Output | DDT in ingresso (materiali ricevuti) | Magazzino |
| `Commerciale/Email Importanti/` | Archive | Mail rilevanti del filo commessa (negoziazione, conferme, decisioni) | PMC |
| `Commerciale/Ordine Cliente/` | Output | PDF dell'ordine cliente firmato | PMC |
| `Commerciale/Preventivo Fattureincloud/` | Output | PDF dei preventivi inviati (FattureInCloud / F.I.C.) | PMC + Direzione (validazione) |
| `Gestionale/` | (al momento poco usato) | Pianificazioni, Gantt, planning sommari | PMC |
| `Tecnico/Documentazione cliente/` | Output | Documenti consegnati al cliente come pacchetto finale | PMC |
| `Tecnico/Fascicolo Tecnico/` | Output | Fascicolo tecnico macchina (Certificazione/Manuale/Schede tecniche) | PMC + Disciplina |
| `Tecnico/Foto/` | Process | Foto durante costruzione e collaudo | Team |
| `Tecnico/Materiali/{a,b,c,d}/` | Process | Liste materiali per disciplina (mecc/fluid/elettr/sw) | Disciplina |
| `Tecnico/Progetto/{a,b,c,d}/` | Process | Sorgenti progettuali per disciplina | Disciplina |
| `Transit/` | Buffer | Parcheggio temporaneo (file da smistare) | Tutti |

**Regola sui file sciolti in root**: la radice della commessa non deve contenere file generici. Eccezioni accettate: `README.md`, `CLAUDE.md`, `LINKS.md` (dichiarate). File `.xlsx`, `.pdf`, `.csv` in root sono **debito tecnico** da estinguere a chiusura commessa (spostare in `Contesto/`, `InterazioniCliente/`, o eliminare se transitori).

---

## 4. `Gestione commessa/` — tracciamento operativo

**Scopo**: ospita i documenti di **conduzione** (non di progetto né commerciali): istanze dei template, rapporti giornalieri, verbali interni, todo list.

**Contenuto canonico**:

| File / cartella | Stato | Cosa è |
|---|---|---|
| `_todo.md` | Vivo | Lista task aperti del PM; aggiornato giornalmente |
| `dod-checklist.md` | Vivo | Istanza di [../procedura/dod-checklist.md](../procedura/dod-checklist.md), spuntato progressivamente |
| `kickoff.md` | Vivo poi congelato | Istanza di [../modelli/kickoff.md](../modelli/kickoff.md), congelata dopo firma |
| `analisi-fattibilita.md` | Storico / opzionale | Compilata ora **nel gestionale** sul Lead (7 sezioni, export PDF). Il file in cartella è solo eventuale export/storico — vedi [../modelli/analisi-fattibilita.md](../modelli/analisi-fattibilita.md) |
| `rapporti/rapporto_YYYYMMDD.md` | Append-only | Rapporti giornalieri (vedi [../modelli/rapporto-giornaliero.md](../modelli/rapporto-giornaliero.md)) |
| `meeting/<data>_<argomento>.md` | Append-only | Verbali riunioni interne |
| `change-requests/CR-NN.md` | Append-only | Change Request (vedi [../modelli/change-request.md](../modelli/change-request.md)) |
| `lesson-learned.md` | Vivo poi congelato | Compilato a chiusura (vedi [../modelli/lesson-learned.md](../modelli/lesson-learned.md)) |

**Regola**: `Gestione commessa/` è il punto unico di ingresso per chi (umano o AI) deve capire **stato attuale** della commessa. Insieme a `README.md` e `CLAUDE.md`, è il primo posto da leggere.

---

## 5. `Contesto/` — input read-only

**Scopo**: contiene l'input immutabile della commessa (capitolato cliente, schemi, datasheet componenti, manuali vendor). È **read-only**: revisioni avvengono per **nuova versione con timestamp**, mai sovrascrittura. Il vecchio resta visibile per tracciabilità.

**Convenzione naming versioni**: `<nome>_v<NN>_<YYYYMMDD>.<ext>` (es. `capitolato_v02_20260403.pdf`).

**Sotto-cartelle**:
- `descrizione-progetto.md` — una pagina di inquadramento, scritta dal PMC al kick-off.
- `capitolato/` — specifiche formali cliente (RFQ, capitolato, normative richieste).
- `schemi/{elettrici,fluidici,meccanici}/` — schemi forniti dal cliente o derivati da esistente.
- `datasheet/{plc,hmi,sensori,attuatori,drive}/` — schede tecniche componenti.
- `manuali/` — manuali vendor di componenti significativi.

**Pattern già consolidato in `669_Muller_Plc/`**: stessa struttura, stessa filosofia. Vedi [../plc/convenzioni-progetto-plc.md §2](../plc/convenzioni-progetto-plc.md).

---

## 6. `_StandardVBA/` — snapshot doc standard

**Scopo**: copia locale dei documenti `elucubrIAmo/` (questo repo) per dare a Claude e ai collaboratori accesso immediato alle convenzioni VBA, **senza dipendere** dalla presenza di `elucubrIAmo/` sulla macchina.

**Regole**:
- **Autorità**: la sorgente è `elucubrIAmo/` (nel Drive condiviso `I - iaRoot/`). Lo snapshot è solo copia.
- **Direzione update**: sorgente → snapshot. **Mai viceversa.**
- Se navigando lo snapshot trovi errori o migliorie: modifica `elucubrIAmo/` e rigenera lo snapshot.
- File `_VERSIONE.md`: data e versione dei file copiati. Aggiornato a ogni sync.

**Contenuto minimo** dello snapshot:
- `README.md` (di `elucubrIAmo/`)
- `procedura/` completa
- `struttura-locale/struttura-commessa.md`
- `discipline/` (le discipline pertinenti alla commessa; per la 0704: tutte le 5)
- `modelli/` (i template usati)
- `plc/` (se la commessa ha disciplina PLC attiva)
- `_VERSIONE.md` con data/versione + cronologia sync

**Aggiornamento**: manualmente all'inizio di ogni commessa nuova o quando il PM vuole allinearsi alle ultime convenzioni. In futuro: skill `/sync-standard-vba` (vedi `plc/convenzioni-progetto-plc.md` §12).

---

## 7. File root: `README.md`, `CLAUDE.md`, `LINKS.md`

### 7.1 `README.md` (obbligatorio)
Scheda commessa human-readable. Una pagina, contenuto stabile. Si aggiorna ai cambiamenti strutturali (profilo, BU, discipline), non al day-to-day.

**Sezioni**:
- Identificazione: numero, cliente, descrizione, BU, profilo workflow, PMC.
- Discipline attive (con flag Progettazione/Manodopera).
- Stato corrente (RoadMapp).
- Riferimenti: link a RoadMapp, repo PLC se esiste, cartella Drive condiviso, ecc.
- Quick links: a `Gestione commessa/_todo.md`, `Gestione commessa/dod-checklist.md`.

### 7.2 `CLAUDE.md` (consigliato)
Istruzioni per Claude Code se si lavora con AI dentro la cartella commessa. Contiene:
- Contesto progetto sintetico.
- Riferimento a `_StandardVBA/` come fonte di convenzioni.
- Regole specifiche di questa commessa (deroghe motivate).
- Regole operative hard ("MAI fare X").
- Indicazione `Gestione commessa/_todo.md` come stato corrente.

Pattern dettagliato in [../plc/convenzioni-progetto-plc.md §7](../plc/convenzioni-progetto-plc.md).

### 7.3 `LINKS.md` (consigliato)
Pointer esterni stabili. Niente cose mutevoli (no TODO, no stato lavoro).

**Sezioni**:
- RoadMapp: URL pagina commessa.
- Google Drive: percorso alla cartella commessa nel Drive condiviso.
- Repo git correlati: PLC (`<roots.github>/<NNNN>_<mnemonico>_plc/`), BE/FE (`<roots.github>/<NNNN>_<mnemonico>/`), HMI esterno (in repo PLC). Path locali risolti dal [profilo macchina](profilo-macchina.md), mai hardcoded.
- Archivio storico: `<roots.vba_data>/G - Archivio/Vph-PLCz/<NNNN>_<mnemonico>/` per i .zap PLC (path indicativo, da confermare con la topologia attuale).
- **Workspace file IDE** (vedi [gruppo-lavoro.md](gruppo-lavoro.md)): elenco dei file workspace generati per ogni postazione (es. `<NNNN>.<host>.code-workspace`). Indicare per chi/quale postazione.
- Contatti: PMC interno, referente cliente, contatti chiave fornitori.

---

## 8. Codifica naming file

**Generale**:
- File markdown: `kebab-case.md` (es. `analisi-fattibilita.md`).
- File template istanziati: stesso nome del template (`kickoff.md`, `verbale-fat.md`).
- Allegati cliente / fornitore: preservare il riferimento originale (protocollo, codice, sigla) **dentro il descrittivo**, eventualmente con prefisso di tipo (vedi §8.1).
- Versioning per file di Contesto/: `<nome>_v<NN>_<YYYYMMDD>.<ext>` (vedi §5).
- Rapporti: `rapporto_YYYYMMDD.md`.
- Meeting: `<YYYYMMDD>_<argomento-breve>.md`.
- CR: `CR-<NN>.md` (numerazione progressiva intra-commessa).

**Date**: sempre formato ISO `YYYYMMDD` (compatto) o `YYYY-MM-DD` (descrittivo).

**Caratteri ammessi**: alfanumerici ASCII + `_`, `-`, `.`. Niente accenti, niente `#`/`§`/`/`. Eccezione: nei nomi di cartella commessa standard si usano spazi (`A - Commesse`, `0704 - ERRE COMPANY - Banco fatica cofani`) per consuetudine VBA.

### 8.1 Prefissi di tipo per documenti ricorsivi di commessa

Convenzione storica VBA, in uso parziale, formalizzata qui per i documenti "lavorabili" della commessa (in particolare in `Contesto/`, `Commerciale/`, `Acquisti/Offerte/`, `Tecnico/Materiali/`).

**Pattern**: `<pp><nnnn> - <descrittivo>.<ext>`
- `pp` = prefisso 2 lettere lowercase, identifica il **tipo** di documento.
- `nnnn` = numero commessa (4 cifre).
- `descrittivo` = descrizione human-readable; può includere protocollo cliente, codice fornitore, data, ecc.

**Prefissi consolidati**:

| Prefisso | Tipo documento | Esempio |
|---|---|---|
| `cc` | Capitolato cliente (input formale dal cliente) | `cc0622 - 2026-80A-SP-003-00.docx` |
| `of` | Offerta (emessa da VBA verso cliente) | `of0622 - tecnico-economica rev2.docx` |
| `pv` | Preventivo (ricevuto da fornitore) | `pv0622 - cella di carico.pdf` |
| `ft` | Foglio tecnico | `ft0622 - layout banco.pdf` |
| `st` | Scheda tecnica (componente) | `st0622 - servomotore xyz.pdf` |
| `co` | Comunicazione (di / verso cliente, fornitore, interna) — il documento, non il vettore (mail/PEC). Tipico: documento di accompagnamento, nota, verbale di sintesi non categorizzabile altrimenti | `co0622 - riepilogo riunione 20240220.docx` |

**Quando si applica**:
- Documenti formali e ricorsivi della commessa (input cliente/fornitore, output VBA, comunicazioni di accompagnamento).
- File appena introdotti o da rinominare in fase di riordino.

**Quando NON si applica**:
- Istanze dei template `Gestione commessa/` (nome canonico fisso: `kickoff.md`, `analisi-fattibilita.md`, `dod-checklist.md`, ecc.).
- Artefatti di sviluppo: codice sorgente, CAD nativi (`.sldprt`, `.dwg`, `.x_t`...), progetti TIA, repo git.
- File legacy preesistenti: non si rinominano retroattivamente — la convenzione si applica ai documenti nuovi o quando un file viene comunque toccato per altro motivo.
- Email-vettore: le `.eml` / `.msg` in `InterazioniCliente/email/` mantengono naming proprio (vedi §5). Il prefisso `co` è per il **documento allegato o di sintesi**, non per il messaggio email in sé.

**Coesistenza col versioning di Contesto/** (§5): per file in `Contesto/` soggetti a revisione, sommare i due pattern come `<prefisso><commessa> - <descrittivo>_v<NN>_<YYYYMMDD>.<ext>`. Esempio per una revisione del capitolato:

```
cc0622 - 2026-80A-SP-003-00_v02_20260710.docx
```

**Estensione**: aggiungere un nuovo prefisso solo dopo accordo collegiale. La lista deve restare corta e mnemonica.

---

## 9. Lead non convertiti (ex "commesse abortite")

**Cambiamento (maggio 2026)**: con la separazione Lead/Commessa, un'opportunità che non va a buon fine **non è più una commessa abortita** ma un **Lead in stato `Respinto` o `Annullato`** (vedi [flusso-commessa.md §2](../procedura/flusso-commessa.md) e [struttura-lead.md §2](struttura-lead.md)). Resta come record nel gestionale e, se aveva una cartella, nel drive `B - Lead`.

Di conseguenza la vecchia cartella `000 - abortite/` in `A - Commesse/` **non serve più** per i lead non convertiti: questi non entrano mai in `A - Commesse`. Resta solo la gestione delle **commesse interrotte dopo l'avvio** (stato `Risoluzione`), per le quali la convenzione di archiviazione è ancora TBD.

---

## 10. Esempio applicato — commessa 0704

La commessa **0704 - ERRE COMPANY - Banco fatica cofani** è la prima istanza piena del paradigma. La struttura sopra descritta verrà applicata alla cartella esistente, **aggiungendo** le nuove cartelle/file, **senza spostare** nulla di quanto già presente. La conformità sarà progressiva.

Stato di partenza (maggio 2026): cartella standard storica `Acquisti/Commerciale/Gestionale/Tecnico/Transit` già presente, vuote o quasi. Aggiunte previste: `Gestione commessa/`, `Contesto/`, `InterazioniCliente/`, `_StandardVBA/`, `README.md`, `CLAUDE.md`, `LINKS.md`.
