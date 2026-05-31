# Convenzioni progetto PLC VBA

**Stato**: bozza 1, aprile 2026.
**Scopo**: definire regole, paradigmi e procedure operative per lavorare su qualunque progetto PLC VBA — nuovo o legacy, con o senza TIA VCI già attivo. Pensato per essere leggibile da umani (sviluppatori nuovi, Paolo, MatTso, Matteo in consultazione) e da AI (Claude Code nei progetti, quando letto via `CLAUDE.md`).

**Correlati**:
- [catalogo-udt-base.md](catalogo-udt-base.md) — dettaglio UDT della libreria standard
- [catalogo-funzioni-base.md](catalogo-funzioni-base.md) — funzioni e function blocks base/pattern, sequenza Main scan cycle
- *catalogo-db-templates.md* (step 3) — DB con struttura standard ma contenuto project-specific
- *anti-pattern-osservati.md* (in arrivo dopo) — trappole osservate con uso non controllato di AI

---

## 1. Gerarchia degli artefatti di standardizzazione

```
elucubrIAmo/plc/                            ← sorgente-verità VBA-wide (questo repo)
    ├── catalogo-udt-base.md
    ├── convenzioni-progetto-plc.md         ← (questo file)
    └── anti-pattern-osservati.md

XXXX_mnemonico_plc/                         ← singolo progetto PLC
    ├── CLAUDE.md                           ← puntatore/estratto di elucubrIAmo/plc
    ├── LINKS.md                            ← legami a cartella commessa (Drive), BE/FE, archivio
    ├── Documentazione/
    │   ├── INDICE.md
    │   ├── 00_Convenzioni.md               ← regole generali + eventuali specializzazioni
    │   └── (capitoli numerati per decine)
    └── ...
```

**Principio**: `elucubrIAmo/plc/` è la sorgente unica. Ogni progetto ne è un'istanza che può specializzare, mai contraddire. Se una regola del progetto contraddice `elucubrIAmo/plc/`, o va promossa a standard o è un bug della specializzazione.

---

## 2. Struttura repo `XXXX_mnemonico_plc/`

Posizione standard: `C:\Documenti\GitHub\XXXX_mnemonico_plc\` sulla macchina di ciascun sviluppatore.

```
XXXX_mnemonico_plc/
├── .claude/                              # configurazione Claude Code (git)
│   ├── settings.json                     # permessi e env condivisi
│   └── skills/                           # skill specifiche del progetto (rare)
├── .git/                                 # repo git
├── .gitignore
├── .vci/                                 # TIA VCI (se abilitato)
├── CLAUDE.md                             # istruzioni AI-facing del progetto
├── LINKS.md                              # indice legami esterni (vedi §6)
├── README.md                             # panoramica progetto human-facing
│
├── TIA_VCI/                              # root del sorgente TIA (PLC + eventuale HMI Siemens)
│   ├── PLC/
│   │   ├── Program blocks/
│   │   ├── PLC data types/
│   │   └── ...
│   └── HMI/                              # solo se HMI nativo TIA (Siemens KTP/Comfort/Unified)
│
├── HMI/                                  # se HMI esterno non-Siemens (Weintek, Pro-face, ecc.)
│   ├── README.md
│   ├── src/                              # sorgente editabile vendor (spesso binary)
│   └── export/                           # tag list CSV, screenshot, macros txt, report
│       ├── tags/
│       ├── schermate/
│       ├── macros/
│       └── report/
│
├── Documentazione/                       # capitoli numerati (vedi §5)
│   ├── INDICE.md                         # output VIVO, prodotto in corsa
│   ├── 00_Convenzioni.md
│   ├── 00_Premesse.md
│   ├── 01_Architettura_Generale.md
│   └── ...
│
├── Contesto/                             # input IMMUTABILE di progetto (read-only)
│   ├── descrizione-progetto.md           # una pagina di inquadramento
│   ├── capitolato/                       # specifiche formali cliente (se presenti)
│   ├── schemi/                           # elettrici, fluidici, meccanici (PDF/DWG)
│   │   ├── elettrici/
│   │   ├── fluidici/                     # olio, aria, acqua, glicole
│   │   └── meccanici/
│   ├── datasheet/                        # schede tecniche componenti
│   │   ├── plc/                          # CPU, moduli IO
│   │   ├── hmi/
│   │   ├── sensori/                      # sonde, finecorsa, pressostati, trasduttori
│   │   ├── attuatori/                    # valvole, motori, pompe
│   │   └── drive/                        # (opzionale, se in commessa)
│   └── manuali/                          # manuali vendor operativi/installazione
│
├── note progetto/                        # lavoro corrente, rapporti giornalieri
│   ├── _todo.md
│   ├── rapporto_YYYYMMDD.md              # uno per giornata di lavoro significativa
│   └── ...
│
├── InterazioniCliente/                   # email importanti, bozze risposte, issue
├── Gestione commessa/                    # relazioni riunioni, coordinamento interno
│
├── _StandardVBA/                         # snapshot documentazione standard (vedi §2b)
│   ├── _VERSIONE.md
│   ├── convenzioni-progetto-plc.md
│   ├── catalogo-udt-base.md
│   ├── catalogo-funzioni-base.md
│   └── ...
│
└── node-red/                             # flow Node-RED se presenti (opzionale)
```

**Ruolo delle cartelle principali** (distinzione importante):

| Cartella | Natura | Chi la popola | Regola |
|---|---|---|---|
| `TIA_VCI/` | Sorgenti PLC + eventuale HMI Siemens nativo | Sviluppatore (via TIA VCI) | Modificata da TIA, mai a mano |
| `HMI/` (opzionale) | Sorgenti HMI esterno (Weintek, Pro-face, ecc.) con export leggibili | Sviluppatore (via tool vendor) | Vedi §2c |
| `Documentazione/` | Output di progetto (doc tecnica prodotta in corsa) | Sviluppatore + Claude | Cresce nel tempo, capitoli strutturati |
| `Contesto/` | Input immutabile (scheme, datasheet, capitolato) | Sviluppatore (una volta) | **Read-only**: revisioni via nuova versione con timestamp, il vecchio resta |
| `note progetto/` | Lavoro corrente (todo, rapporti giornalieri) | Sviluppatore | Muta spesso, informale |
| `InterazioniCliente/` | Correspondence viva | Sviluppatore | Email/bozze/issue |
| `Gestione commessa/` | Coordinamento interno | Team | Meeting, gantt (se usati) |
| `_StandardVBA/` | Copia riferimenti standard (snapshot) | Aggiornamento scheduled | **Read-only**: sorgente è `elucubrIAmo/plc/` |

**Commento sui file sciolti in root**: nel 0645 ci sono xlsx/pdf/csv sparsi alla radice. Accettabili come parcheggio temporaneo ma **alla fine della commessa devono trovare casa** (in `Contesto/*`, `InterazioniCliente/`, o eliminati se transitori).

### 2b. `_StandardVBA/` — snapshot della documentazione standard

**Perché esiste**: la sorgente di verità della documentazione standard VBA è in `elucubrIAmo/plc/` (questo percorso, su macchina Roberto). Finché non c'è un sistema di condivisione formale (submodule git, libreria, pacchetto), ogni progetto porta con sé una **copia locale** dei file standard. Così la chat Claude aperta su un repo `XXXX_plc/` ha immediato accesso alla documentazione, anche su macchine dove `elucubrIAmo/` non esiste.

**Regole snapshot**:
- **Autorità**: sorgente `elucubrIAmo/plc/`. Lo snapshot è solo copia.
- **Direzione update**: sorgente → snapshot. **Mai viceversa.**
- **Se trovi errori/migliorie navigando lo snapshot**: modifica in `elucubrIAmo/plc/`, poi rigenera lo snapshot.
- **File `_VERSIONE.md`**: riporta data e versione dei file copiati. Aggiornato ad ogni sync.
- **Aggiornamento**: manualmente all'inizio di ogni commessa nuova o quando lo sviluppatore vuole allinearsi alle ultime convenzioni. In futuro: skill `/sync-standard-vba` che automatizza.

**Contenuto minimo** (aggiornabile all'evoluzione dei documenti in elucubrIAmo):
- `_VERSIONE.md` — data/versione + istruzioni update
- `convenzioni-progetto-plc.md`
- `catalogo-udt-base.md`
- `catalogo-funzioni-base.md`
- `catalogo-db-templates.md` (quando esisterà)
- `anti-pattern-osservati.md` (quando esisterà)

**Migrazione futura**: quando implementeremo il sistema di condivisione vero (git submodule, libreria, pacchetto), `_StandardVBA/` verrà rimpiazzato o diventerà un symlink/submodule. Per ora: copia consapevole.

### 2c. Progetti HMI esterni (non-Siemens)

Se l'HMI è nativo Siemens (KTP, Comfort, Unified), vive **dentro** il TIA project → cartella `TIA_VCI/HMI/`. Versioning tramite TIA VCI insieme al PLC. Nessuna specialità.

Se l'HMI è **esterno** (Weintek EasyBuilder, Pro-face GP-Pro EX, altri), il suo tooling è separato da TIA. Convenzione VBA:

- **Cartella parallela** al TIA project: `HMI/` allo stesso livello di `TIA_VCI/` nel repo di commessa.
- **Naming**: fisso `HMI/` (astratto, cross-commessa). Se un futuro banco avesse due HMI esterni di vendor diversi coesistenti, si potrebbero distinguere con sottocartelle interne (`HMI/Weintek/`, `HMI/Proface/`). Caso raro.
- **Struttura interna**: `src/` per il sorgente editabile (tipicamente binary proprietary), `export/` per materiale leggibile da umani e AI.
- **Ambiguità risolta**: `TIA_VCI/HMI/` = HMI Siemens nativo (se presente); `HMI/` a root = HMI esterno. Path completo distingue sempre.

**Sotto-cartelle di `export/`** (una per categoria di export utile):

| Cartella | Contenuto | Scopo |
|---|---|---|
| `tags/` | Address Tag Library esportata (CSV/XML) | Corrispondenza tag HMI ↔ indirizzi PLC (holding register, aree S7, ecc.). Leggibile da Claude per verifiche di coerenza. |
| `schermate/` | PNG delle schermate principali | Documentazione visiva dell'interfaccia. Leggibile da Claude come immagini. |
| `macros/` | Script Macro copiati come `.txt` | Logiche lato HMI fuori dallo spazio visuale. |
| `report/` | PDF/HTML generati dal tool HMI | Report completo del progetto HMI. |

**Regola operativa**:
- Il sorgente binario (`.emtp` per Weintek, `.prx` per Pro-face, ecc.) **va committato** per tracciabilità, anche se non produce diff leggibile.
- Gli export `export/` **vanno rigenerati** prima di ogni commit significativo del sorgente HMI. Senza questa disciplina, Claude lavora su immagine obsoleta dell'HMI.
- **Minimo vitale** per l'accessibilità a Claude: `tags/` aggiornato + `schermate/` con le viste principali.

**Comunicazione PLC↔HMI**: tipicamente Modbus TCP (PLC slave). Il layout del server lato PLC è in `dbComm` → la corrispondenza tag HMI ↔ `dbComm` va documentata in `Documentazione/3X_Comunicazione_<vendor>.md`. Ogni modifica di `dbComm` implica aggiornamento di `export/tags/` (e viceversa).

---

## 3. Struttura `<TIA-project>/PLC/Program blocks/`

Già stabile nel 0645 e adottata come standard:

```
Program blocks/
├── 10-Sistema/            # OB di sistema, Main, startup, gestione errori
├── 20-IO/                 # lettura DI, scrittura DQ, lettura AI, scrittura AO
├── 30-Comunicazione/      # Modbus (master/slave), PC↔PLC, interfaccia esterna
├── 40-Regolazione/        # PID, PWM, termoregolazione, gestione motori, valvole proporzionali
├── 50-Funzioni/           # state machine applicative, logica ciclo, funzioni helper
├── 60-Mapping/            # fisico↔virtuale, proiezioni, mappatura canali
├── 70-DB/                 # data block principali (dbB, dbAIs, dbComm, ecc.)
├── 80-Init/               # inizializzazione, taratura, valori di default
└── 90-Archivio/           # blocchi obsoleti non cancellati per riferimento storico
```

**Regole struttura**:
- Numerazione **per decine** (stile BASIC line-number): consente inserimento di sottogruppi futuri (es. `45-TerModulare/`) senza rinumerare.
- Un blocco risiede in **una sola** cartella, scelta dal ruolo **principale**. Se dubbio, prevale la cartella col numero più basso coerente.
- `90-Archivio/` non è una pattumiera: contiene blocchi tenuti volutamente per confronto o rollback. I blocchi davvero morti vanno cancellati.

### Marker scope sui blocchi codice (4 marker)

A differenza degli UDT (dove lo scope è esplicitato dalla cartella `Base/Esterni/Progetto`), per il codice la cartella esprime il **ruolo** (Sistema, IO, Comunicazione…). Lo scope si segna **inline nel filename**:

| Marker | Significato | Vive in | Modificabile come? |
|---|---|---|---|
| **(nessuno)** | Base secca, importata invariata dal modello | Modello + progetti | Mai modificare nel progetto (modifiche tornano al modello via promozione) |
| **`#pat`** | Pattern/template puro | **Solo modello** | Modificabile solo nel modello |
| **`#paj`** | Ex-pattern adattato in un progetto specifico | **Solo progetti** | Modificabile rispettando le convenzioni del pattern padre |
| **`#prj`** | Specifico al progetto, nato e morto qui | **Solo progetti** | Modificabile liberamente |

**Le 3 transizioni**:
1. `#pat → #paj` — quando importi un pattern dal modello in un progetto e lo adatti (via standard).
2. `#prj → #paj` — promozione retro, quando un file project-specific viene riconosciuto come istanza di un pattern del modello.
3. `#paj → #prj` — degradazione, quando un ex-pattern diverge talmente dal padre da perdere traceability. Da fare consapevolmente.

**Perché il codice ha `#paj` e gli UDT no**: per gli UDT lo scope è già nella cartella (`Base/Progetto`); per il codice, dove la cartella esprime ruolo, serve un marker inline. Asimmetria voluta.

### Codice esterno (vendor)

Codice da librerie/esempi vendor (es. Siemens MB master, OB di error management standard) **non riceve marker** ma vive in **sotto-cartelle dedicate** che lo identificano per origine (es. `30-Comunicazione/MB master/`, `10-Sistema/ErrorMgt/`). Convenzione: una sotto-cartella esplicita dentro la cartella numerica di pertinenza.

### Infrastruttura standard VBA

Esiste un piccolo set di **DB e costanti VBA-wide** che è considerato **infrastruttura sempre presente** in ogni progetto. Le funzioni `f_` astratte possono dipendere da questa infrastruttura senza perdere lo status di "astratte" (no marker). Questo permette di scrivere helper che operano su `dbA`/`dbAIs`/`dbB` senza esplodere in marker `#pat` ovunque.

| Risorsa | Tipo | Cosa è |
|---|---|---|
| `dbA` | DB | Vettori IO globali: `i.i[]` (digitali in), `q.dev[]` (digitali out con `t_dev`), `TMcAlive` (clock di riferimento), `id`/`qd` (mnemonici indici) |
| `dbAIs` | DB | Allarmi: `a[]` array `t_alStr`, flag globali (`flSpiaOn`, `flSonoro`, `flSpiaBlink`, `ack`, `reset`, `mute`), indici nominativi |
| `dbB` | DB | Dati di banco: `glob`, `cOlio`/`cGlic`/`cTras` o equivalenti. Nome stabile, **struttura interna varia per banco** (qui sta il limite — vedi sotto) |
| `cfgN*` | costanti | Dimensioni di sistema: `cfgNDi`, `cfgNDo`, `cfgNAi`, `cfgNAlarm`, `cfgNDiW`, `cfgNDoW`, `cfgNAlarmW`, ecc. |
| `Clock_1Hz` (e simili) | system tag | Memory area Siemens convenzionata per clock 1 Hz. Configurabile in CPU ma standard de-facto VBA |

**Limite del principio**: una `f_` può accedere a `dbB` solo per i campi che fanno parte della **convenzione di interfaccia VBA standard** — cioè campi che esistono in **qualunque** istanza di `dbB` con la stessa semantica. Tipico: i bit di `dbB.glob.cmd.req2plc.*` (perché `t_cmd2plc#pat` ha bit semantici standard una volta adattati). Non tipico: tag fisici banco-specifici tipo `dbB.cOlio.tr.tr.YVuRAFFTRAS_2040`.

Quando il confine è ambiguo, la regola operativa: **se la funzione si rompe quando porti dbB in un altro banco, allora va marcata `#pat`**. Se invece resiste perché tocca solo campi di convenzione standard, può restare astratta.

Dettaglio funzioni → [catalogo-funzioni-base.md](catalogo-funzioni-base.md).

---

## 4. Struttura `<TIA-project>/PLC/PLC data types/`

Gerarchia a 3 folder di primo livello (PascalCase, ordine alfabetico naturale):

```
PLC data types/
├── Base/                          # VBA-owned, riusabile ovunque (candidato libreria TIA VBA)
│   ├── 00-fondamentali/           # t_ph, t_phHiStep, t_dev, t_anyPtr, t_cBitInt
│   ├── 10-IO_digitali/            # t_di, t_dq, t_di_def#pat, t_dq_def#pat, t_hsCnt
│   ├── 20-IO_analogici/           # t_aiCh, t_aoCh, t_anChCfg/Ao, t_aiMbCh, t_aoMbCh,
│   │                              # t_aiProcCComm#pat, t_aiProcCirc#pat
│   ├── 30-regolazione/            # t_PIDstr, t_pwm, t_soc
│   ├── 40-allarmi_monitoraggio/   # t_alStr, t_bitCfgAl, t_monEv#pat, t_monEvs#pat
│   ├── 50-comandi/                # t_cmd2plc#pat, t_cmd2pc#pat, t_cmdSvc#pat,
│   │                              # t_manCmd#pat, t_gCmd, t_rstSkipMask#pat
│   ├── 60-setpoint_base/          # t_spCfg/Grp/Liv/Tm/To#pat, t_stepSp, t_stepSpStatus
│   ├── 70-termoregolazione/       # t_trBase, t_tRegFl
│   └── A0-devices/                # wrapper VBA standard per device di vendor abituali
│       └── <Vendor>/              # es. DeltaDrives/, SiemensVFD/, ...
│
├── Esterni/                       # terze parti (Siemens, altri vendor), spesso protetti, non-VCI
│   └── Modbus RTU/                # da libreria/esempio Siemens (Data_for_Master, Data_Slave)
│
└── Progetto/                      # specifico commessa/hardware (NON viene mai duplicato)
    ├── Macro/                     # aggregati dbB-like specifici (t_dbBGlob, t_dbBCircuito, t_dbBTras...)
    ├── SetPoints/                 # setpoint specifici progetto (t_spCircuito, t_spTras...)
    ├── Test/                      # ciclo prova specifico (t_test, t_testHeader, t_testStep...)
    ├── Vari/                      # UDT specifici senza famiglia (es. t_port)
    └── VirtCirc/                  # circuito virtuale (vCirc) — pattern ripescabile
```

### Regole struttura

- **Scope esplicito al primo livello**: `Base/` = sacro (modifiche si ripercuotono ovunque), `Progetto/` = libero, `Esterni/` = non toccare.
- **Numerazione per decine in `Base/`** (00, 10, 20…, A0): consente inserimento di sottogruppi futuri. `A0` (esadecimale) per categorie "extra" oltre il range numerico naturale (riserva 80, 90 per espansioni base future).
- **In `Progetto/` prevale il nome parlante**, non la numerazione.
- **A0-devices/<Vendor>/**: wrapper VBA per famiglie di device che VBA usa abitualmente. Una sottocartella per famiglia. Il **pattern di decomposizione** raccomandato è `ph + eng + pts + naif + sp + onLine` (vedi catalogo UDT).

### Marker `#pat` su UDT

UDT con suffisso `#pat` nel nome = **scheletro/template da adattare** prima dell'uso operativo nel progetto:
- bitfield comandi con bit semantici banco-specifici
- aggregati AI con canali del banco specifico
- UDT con indici interni da inizializzare
- definizioni canali DI/DQ con nomi banco
- setpoint pattern con default da rivedere

**Ciclo di vita**:
- Nel **modello/template VBA** (futuro `elucubrIAmo/plc/template-progetto/`) → `#pat` mantenuto.
- Nel **progetto adattato** → `#pat` rimosso quando l'adattamento è completato.
- Eccezione: UDT con campi placeholder (`flN`, `auxCmd_NN`) per espansione futura mantengono i placeholder, ma il `#pat` esterno può essere rimosso.

**Caratteri TIA-safe**: `#` è ASCII, supportato sia da Windows NTFS sia da TIA (auto-quoting). TIA accetterebbe anche `°`, `§`, `/`, ma `/` rompe il sync filesystem; `°`/`§` introducono problemi di encoding in pipeline esterne.

### Convenzioni naming cartelle

- **PascalCase** per le cartelle di gruppo (`Base`, `Esterni`, `Progetto`, `Macro`, `SetPoints`, `VirtCirc`).
- **lowercase + underscore** per le sottocartelle numerate (`00-fondamentali`, `40-allarmi_monitoraggio`, `60-setpoint_base`).
- **PascalCase per `<Vendor>`** sotto `A0-devices/` (`DeltaDrives`, `SiemensVFD`).
- **Evitare due maiuscole consecutive** nei nomi cartelle (preferito `IO_digitali` a `IODigitali`).

### Cosa va in `Progetto/` (regola di Roberto)

> *"Tutto ciò che è in Progetto non va duplicato per nulla, essendo specifico di quell'impianto."*

`Progetto/` non viene mai copiato fra commesse. **Tendenzialmente tutto ciò che sta sotto `Progetto/` viene eliminato** durante la conversione/creazione di una nuova commessa a partire dal modello.

**Eccezione consentita**: il **modello progetto** (futuro template VBA) può contenere in `Progetto/` un **set minimo di esempi funzionanti e autoesplicativi** — l'obiettivo è che il template sia immediatamente compilabile e leggibile invece che uno scheletro vuoto e silenzioso. Sono ammessi anche **rari casi di coerenza d'uso o componenti standardizzati del banco** che ricorrono in praticamente tutti i nostri impianti (da identificare a posteriori, non a tavolino).

Quando si parte dal template per una nuova commessa: tutto `Progetto/` va riletto; quello che non si applica si **elimina**, quello che si applica si **adatta**. Niente è dato per scontato.

Dettaglio UDT → [catalogo-udt-base.md](catalogo-udt-base.md).

---

## 5. Numerazione capitoli `Documentazione/`

**Principio**: allineare i range dei capitoli ai numeri delle cartelle `Program blocks/`. Chi legge il capitolo 40-qualcosa sa di parlare di regolazione; il codice corrispondente è in `40-Regolazione/`.

```
00-09  Convenzioni, premesse, panoramica architettura, capitolato banco, glossario
10-19  Sistema (Main, OB, startup, error management)                  [↔ 10-Sistema]
20-29  I/O (DI/DQ, AI/AO, riparametrazioni, tarature)                 [↔ 20-IO]
30-39  Comunicazione (Modbus drive, PC↔PLC, Node-RED, handshake BE/FE) [↔ 30-Comunicazione]
40-49  Regolazione (PID, PWM, termoregolazione, motori, valvole prop.) [↔ 40-Regolazione]
50-59  Funzioni applicative e logica ciclo (state machines, ciclo prova) [↔ 50-Funzioni]
60-69  Mapping fisico↔logico, layer di astrazione                     [↔ 60-Mapping]
70-79  Data Blocks (dbB, dbAIs, dbComm, strutture dati)               [↔ 70-DB]
80-89  Init, taratura, messa in servizio                              [↔ 80-Init]
90-99  Archivio documentale, direzioni abbandonate, troubleshooting storico [↔ 90-Archivio]
```

### Regole

- Ogni capitolo è un file `NN_Titolo.md` (es. `42_Termoregolazione.md`).
- Decine **grandi** (range ampio in una decina) lecite: `42`, `43`, `44` tutte in "regolazione" va bene.
- Decine **saltate** altrettanto lecite: se un progetto non ha Mapping, il range 60-69 è vuoto — non rinumerare gli altri per chiudere il buco.
- Sotto-documenti con lettere (`42a`, `42b`) ammessi per approfondimenti a blocchi, ma preferire un documento singolo ben strutturato.
- `INDICE.md` obbligatorio: è la mappa letta per prima da umani e AI.
- `00_Convenzioni.md` obbligatorio: contiene il link a questo file + eventuali specializzazioni di progetto.

---

## 6. File `LINKS.md` — template

Unico file a livello radice del repo `_plc`. Scopo: rendere esplicite le relazioni verso l'esterno senza ricorrere a symlink fragili.

```markdown
# Legami esterni — XXXX_mnemonico_plc

**Commessa**: XXXX — <nome esteso>
**Descrizione sintetica**: <una riga>

## Cartella commessa (Google Drive)
- Percorso: `G:\Drive condivisi\A - Commesse\XXXX - <nome>`
- Contenuto: Acquisti, Commerciale, Tecnico/Progetto/d.software (questo repo è la sorgente canonica del software), …

## Repo fratelli
- Backend/Frontend: `C:\Documenti\GitHub\XXXX_mnemonico\` (BE/FE gestionale)
- Eventuale C# legacy: `C:\Documenti\GitHub\rNNNN-<nome>\` (archivio, non più attivo)

## Archivio .zap storico
- `C:\Documenti\GitHub\Vph-PLCz\XXXX_<mnemonico>\`

## Documentazione vendor in questo repo
- `docs-vendor/` (opzionale)
- link esterni a datasheet pubblici, se non archiviati localmente

## Contatti referenti
- Software PLC: Roberto Visca
- BE/FE: Matteo Visca
- Elettrico: <nome>
- Cliente: <nome ref., mail, tel>
```

**Regole**: `LINKS.md` non contiene roba cangiante (niente TODO, niente stato del lavoro). Se una relazione cambia, si aggiorna qui e via. È l'ancora di contesto.

---

## 7. File `CLAUDE.md` — template progetto PLC

Alla radice del repo `_plc`. È il file che Claude Code legge per primo in ogni sessione. Scopo: dire a Claude cosa deve sapere per essere utile **subito**.

```markdown
# CLAUDE.md — XXXX_mnemonico_plc

## Contesto progetto
Banco prova <tipo>, Siemens S7-<modello>, drive <vendor>, comunicazione <protocolli>, orchestrato da BE/FE <link al repo>. Dettagli in `README.md` e `LINKS.md`.

## Regole e convenzioni
Seguono le convenzioni VBA standard. Sorgente di verità: [elucubrIAmo/plc/](<percorso relativo o assoluto a elucubrIAmo>).
- Struttura cartelle codice e dati: vedi `convenzioni-progetto-plc.md` §3 e §4.
- Naming e UM: vedi `convenzioni-progetto-plc.md` §8.
- Catalogo UDT base: vedi `catalogo-udt-base.md`.
- **Prima di proporre grandi astrazioni, consulta `anti-pattern-osservati.md`.**

## Regole specifiche di questo progetto
<eventuali deroghe o specializzazioni, motivate>

## Regole operative hard
<cose da non toccare mai, es. "MAI correggere offset 1300/HR650 di dbComm zona lettura">

## Rapporti giornalieri
A fine sessione di lavoro significativa, creare `note progetto/rapporto_YYYYMMDD.md` col formato §9 delle convenzioni.

## File chiave
- `<TIA-project>/PLC/Program blocks/10-Sistema/Main.scl` — orchestrazione
- `<TIA-project>/PLC/Program blocks/70-DB/dbB.db` — DB di banco (dati runtime)
- `Documentazione/INDICE.md` — mappa documentazione
- `note progetto/_todo.md` — lavoro in corso
```

**Regole**: `CLAUDE.md` corto e stabile. Le informazioni che cambiano spesso (lavoro in corso, aperti) stanno in `note progetto/_todo.md`, non qui.

---

## 8. Naming conventions (sintesi)

### Prefissi

| Prefisso | Ruolo | Esempi |
|---|---|---|
| `t_` | tipo dato (UDT) | `t_ph`, `t_di` |
| `fc` | function block chiamato ciclicamente | `fcDiRd`, `fcAoWr` |
| `f_` | funzione pura / helper | `f_pid`, `f_pwm` |
| `db` | data block | `dbB`, `dbAIs` |
| `cfg` | costante utente di configurazione (dimensioni) | `cfgNDi`, `cfgNDo` |
| `CP` | costante utente indice canale mnemonico | `CPdiXXX`, `CPaoYYY` |
| `pv` / `sp` | process value / setpoint | `pvPresMonte`, `spTemp` |
| `cmd` / `fl` | comando / flag | `cmdStart`, `flDone` |

### Unità di misura nelle variabili numeriche (3 pattern)

**1. Suffisso dopo `_`** (preferito per nuovi codici):
- Tempo: `_c` centisecondi, `_s` secondi, `_ms` millisecondi, `_h` ore.
- Grandezze fisiche: `_V`, `_A`, `_Hz`, `_KW`, `_grad`, `_rpm`, …

**2. Embedding nel nome** (storico, in uso): pattern `tm[UM][Semantica]` per le variabili tempo.
- `tmIn` = timestamp (diff contro clock corrente, nessuna UM dichiarata).
- `TMcElap` = elapsed in **c**entisecondi.
- `tmsRun` = run time in **s**econdi.
- `tmcOn` / `tmcOff` = centisecondi ON/OFF.

**3. Suffisso moltiplicatore SI** (per scaling raw points, vedi `t_DeltaDrvPts`):
- `_cHz` = centi Hz (×0.01), `_dV` = deci V (×0.1), `_hW` = hecto W (×100).
- Esempi storici: `mA`, `cC` (centesimi di centigrado), `dHz`, `cA`, `dV`.
- `xCento` (per cento, valore × 100), `xMille` (×1000).

Lettera dopo `_` o prefisso = moltiplicatore decimale SI (`m` milli, `c` centi, `d` deci, `h` hecto, `K` kilo, `D` deca). `x` = "per", seguito dal divisore implicito.

**Quando serve**: il pattern moltiplicatore SI ha senso **quando si passano interi in fixed point** (tipico delle comunicazioni Modbus/registri PLC dove un Real costerebbe 2 register e perderebbe precisione/range nei prodotti vettore). Tipico esempio: drive che espongono frequenza come Int in centi-Hz per stare in 16 bit.

**Quando NON serve**: il **nuovo paradigma di comunicazione VBA** (PC↔PLC via S7TCP) passa **Real (float)** direttamente, **annullando la necessità del fixed point**. Per i campi engineering (`*Eng`) usa Real con UM in chiaro nel suffisso (`_Hz`, `_V`, `_KW`); il moltiplicatore SI resta solo per il livello `*Pts` (raw register dei device esterni che parlano fixed point come i drive Modbus).

### Naming bit con indice hex inline (convenzione VBA evoluta)

Pattern per bitfield: `<prefisso2-3char><indiceHexNelGruppo><continuazioneSemantica>`

Esempi (da `t_cmd2plc#pat`):
- `al00armRst` = bit 0 di Word 0, allarme reset
- `ge02nRst` = bit 2 di Word 0, generale reset
- `co0Enfirm` = bit 14 (`E`) di Word 0, conferma
- `ol10io` = bit 0 di Word 1, selezione olio

**Vantaggi**: leggibile come parola, traccia immediata della posizione bit, ammette espansione semantica.

**Bit placeholder**: `flN` o `<prefisso>NN` per bit non ancora semanticizzati. Quando rinomini, mantieni la posizione hex.

**Sostituisce** la convenzione `boXYolFlag` proposta nelle prime bozze per `t_cBitInt`. Per `t_cBitIntX` (32 bit) la convenzione è da decidere alla prima occasione d'uso reale.

### Pattern decomposizione device esterno

Per qualunque wrapper VBA di device esterno (drive, strumento, sensore intelligente):

```
t_<Vendor><Famiglia> {
   ph     : t_ph;                                // stato SM driver
   eng    : t_<Vendor><Famiglia>Eng;             // valori engineering
   pts    : t_<Vendor><Famiglia>Pts;             // valori raw (con suffissi UM SI)
   naif   : t_<Vendor><Famiglia>Naif;            // status non decodificato, informazioni native
   sp     : t_<Vendor><Famiglia>Sp;              // setpoint e comandi
   onLine : Bool;
}
```

Esempio realizzato: `t_DeltaDrv` in `Base/A0-devices/DeltaDrives/`.

### Pattern contenitore stabile + sub-struct pattern

I contenitori che aggregano UDT `#pat` hanno **nome stabile** (no `#pat` esterno). Esempio: `t_gCmd` aggrega `t_cmd2pc#pat`/`t_cmd2plc#pat`/`t_cmdSvc#pat` ma resta `t_gCmd`. Garantisce stabilità delle interfacce di alto livello.

### Famiglia Gefran (legacy documentato)

Vedi [catalogo-udt-base.md §Convenzioni emerse](catalogo-udt-base.md). Nomi `soc`, `Tmist`/`Tmast`/`ItMist`/`ItMast`, `minua`/`maxua`/`fsua` mantenuti per continuità, documentati nel `00_Convenzioni.md` di progetto. **Non introdurre nomi nuovi in questa famiglia** — i nomi VBA moderni seguono i 3 pattern UM moderni (vedi sopra).

### Attributo `S7_SetPoint`

Su campi che non devono essere modificabili a runtime (stato interno SM, storico): `{ S7_SetPoint := 'False' }`. Su campi che sono setpoint dall'HMI: `{ S7_SetPoint := 'True' }` esplicito. Pattern obbligatorio per i campi non-banali.

---

## 9. Rapporto giornaliero — formato

Un file per giornata di lavoro significativa, in `note progetto/rapporto_YYYYMMDD.md`. Formato con **frontmatter YAML** per essere machine-readable (preparazione al futuro ponte col gestionale VBA, vedi [discussione futura con Matteo](<memoria interna>)).

```markdown
---
data: 2026-04-18
commessa: "XXXX"
autore: <email o sigla>
durata_h: 3.5
tag: [plc, documentazione, <altri>]
repo: XXXX_mnemonico_plc
---

# Rapporto YYYY-MM-DD — <titolo sintetico>

## Attività svolte
- ...

## Problemi aperti / da riprendere
- ...

## Decisioni prese
- ...

## Riferimenti
- Commit: `<hash>`, `<hash>`
- Documenti toccati: `Documentazione/42_...md`
```

**Regola**: `durata_h` è tempo **effettivo** sulla commessa, arrotondato a 0.25h. Il testo è quello che copierai poi nel gestionale come descrizione della registrazione ore — scrivilo pensando al lettore (cliente? te stesso tra 6 mesi? il revisore?). **Non** scriverlo come un diario narrativo.

---

## 9b. Modello progetto VBA (template di kickoff)

**Concetto** introdotto da Roberto 2026-04-19: una **ossatura pulita di progetto PLC** (da costruire in `elucubrIAmo/plc/template-progetto/` dopo stabilizzazione del catalogo) contenente:

- Tutto `Base/` (UDT secchi + UDT pattern marcati `#pat`)
- Tutto `Esterni/` standard
- `Progetto/` **vuoto** (o con sola struttura cartelle e placeholder)
- Skeleton dei blocchi codice (`Main`, `fcDiRd`, `fcDoWr`, ecc.) con placeholder dove necessari
- `CLAUDE.md` template, `LINKS.md` template, `Documentazione/INDICE.md` con capitoli pre-strutturati
- Skill di kickoff postazione preconfigurate

**Uso**: si copia/clona il template per ogni nuova commessa o per revitalizzare una commessa legacy. Sostituisce l'anti-pattern attuale (clonare un progetto esistente, rinominare, modificare puntualmente).

**Ciclo di adattamento del template per un nuovo progetto**:
1. Copia template in `XXXX_mnemonico_plc/`.
2. Rinomina TIA project folder (`<TIA-project>/`).
3. Apri in TIA, applica versione PLC corretta, configura HW.
4. Per ciascun UDT `#pat` in `Base/` che ti serve adattare al banco specifico:
   - rinomina/personalizza i campi placeholder
   - rimuovi suffisso `#pat`
5. Popola `Progetto/` con UDT specifici del banco.
6. Compila `Documentazione/00_Convenzioni.md` con eventuali deroghe/specializzazioni.
7. Aggiorna `LINKS.md` con i percorsi reali (cartella commessa su Drive, repo BE/FE, archivio).

**Manutenzione del template**: ogni volta che si scopre un pattern nuovo davvero standard (un UDT promosso da `Progetto/` a `Base/` in più di una commessa), si propaga nel template.

---

## 10. Procedura per progetti legacy (senza VCI TIA)

Caso tipico: si deve mettere mano a una commessa vecchia dove il PLC esisteva solo come `.zap` in `Vph-PLCz` e magari esploso in `c:\Documenti\s7ws\`. Nessuna struttura repo, nessuna documentazione, nessun CLAUDE.md.

**Quadro versioni TIA in VBA (al 2026-04)**: i progetti spaziano da **TIA 15.1 a 20**, passando per 19. Sulle macchine si tende a installare **una sola versione di TIA condivisa** — pratica "poco onesta" ma comoda, che va a sbattere quando un legacy era stato fatto su una versione che oggi non c'è più o quando in officina i banchi sono ancora alla loro versione originale.

**Procedura**:

1. **Classifica l'esigenza**: solo consultazione/documentazione, oppure modifiche reali?

2. **Se solo consultazione**:
   - Apri `.zap` in TIA (versione compatibile, anche più recente di quella originale) → export sorgenti → snapshot in `<commessa>_plc_snapshot/` (no repo git).
   - Crea minimo `CLAUDE.md` + `LINKS.md` in quella cartella.
   - Claude legge sorgenti, scrive documentazione, non tocca il PLC reale. Rischio nullo.

3. **Se modifiche reali**:
   - **Aggiornamento versione TIA = tassativo**. Allinea il progetto alla **versione TIA corrente VBA** (la più recente fra quelle in uso attivo). I progetti spezzettati su 5+ versioni TIA sono fonte costante di scivoloni. La migrazione è solitamente proposta in automatico all'apertura ma **va testata in laboratorio prima del deploy** (blocchi protetti, librerie esterne incompatibili, variazioni comportamentali).
   - **Aggiornamento firmware CPU PLC = tassativo dove necessario**. Se il TIA aggiornato richiede un firmware CPU minimo non disponibile sul PLC fisico, va pianificato downtime e aggiornamento prima del deploy.
   - **Retrofit VCI** (preferita quando fattibile): abilita version control nel TIA aperto, primo sync, poi il progetto si comporta come i nuovi. Costo una tantum.
   - **Snapshot + git** (se VCI non praticabile): apri in TIA, export sorgenti, `git init`, primo commit. Hai storia modifiche sui sorgenti anche senza VCI.

4. **In entrambi i casi (consultazione e modifiche)**: applica la struttura repo `XXXX_mnemonico_plc/` di §2, copia template `CLAUDE.md` (§7) e `LINKS.md` (§6), crea `Documentazione/INDICE.md` e `Documentazione/00_Convenzioni.md`. Il progetto legacy diventa "come se fosse nuovo" dal punto di vista di Claude.

5. **Conversione al modello VBA** (opzionale): dopo aver portato il legacy alla versione TIA corrente e in struttura repo standard, valutare se conviene migrare le UDT al catalogo standard (`Base/`/`Esterni/`/`Progetto/`). Operazione costosa, va fatta solo se il legacy entra in manutenzione attiva o evolutiva.

**Regole**:
- **Non mescolare retrofit con modifiche operative urgenti**. Prima retrofit (o snapshot) + aggiornamento TIA, poi modifiche. Fare insieme produce commit caotici e attribuzioni sbagliate di "chi ha rotto cosa".
- **Mai aggiornare TIA in produzione senza test in laboratorio**.

---

## 11. Setup nuova postazione / nuovo sviluppatore

Quando un nuovo sviluppatore VBA (o una nuova macchina dello stesso sviluppatore) deve mettersi al lavoro su una commessa PLC:

1. **Prerequisiti software**: TIA Portal (versione compatibile col progetto), VS Code, estensione Claude Code, git, accesso ai Drive condivisi Google VBA.
2. **Autenticazione Claude Code**: `claude --login` la prima volta.
3. **Clone del repo PLC**: `git clone <url> C:\Documenti\GitHub\XXXX_mnemonico_plc`.
   - Con il clone arrivano `CLAUDE.md`, `.claude/settings.json`, `.claude/skills/` — Claude Code è già configurato.
   - Con il clone arriva anche `_StandardVBA/` — la documentazione standard è **disponibile localmente** senza necessità di avere `elucubrIAmo/` installato sulla macchina.
4. **Apri il repo in VS Code + Claude Code**: Claude legge `CLAUDE.md`, carica contesto. Chi lavora ha subito il quadro.
5. **TIA**: apri il progetto TIA dalla cartella `<TIA-project>/` del repo. Se VCI attivo, sincronizza; altrimenti estrai `.zap` dall'archivio citato in `LINKS.md`.
6. **Leggi** `note progetto/_todo.md` per capire lo stato corrente.
7. **Se ti serve quadro più ampio**: `Contesto/` per input progetto, `_StandardVBA/` per convenzioni e cataloghi di riferimento.

Questo processo è candidato a diventare la **skill `/kickoff-postazione-plc`** (verifica stato + propone-chiede-esegue passo per passo). Da implementare dopo stabilizzazione convenzioni.

---

## 12. Questioni aperte

1. **Cartella `docs-vendor/`**: superata dalla nuova `Contesto/datasheet/` e `Contesto/manuali/` (vedi §2). Il 0645 avrà un retrofit quando si procederà ad allinearlo allo standard.
2. **`node-red/` al root o dentro `TIA_VCI/`**? Nel 0645 è alla radice. Se i flow sono accessori alla macchina va bene; se sono parte integrante del controllo, potrebbero stare più vicini.
3. **Sincronizzazione personali skills/settings tra macchine di Roberto**: quali usare come personali globali (`~/.claude/skills/`) vs per-progetto (`.claude/skills/`)? Proposta: tutto ciò che è VBA-wide sta in `elucubrIAmo/plc/skills/` e viene installato come simlink/copy in `~/.claude/skills/`. Decisione da rifinire quando scriveremo le prime skill vere.
4. **Migrazione 0645 al nuovo standard**: quando e come? Mia proposta: non adesso (0645 è in consegna attiva); fare la revisione con la seconda commessa che parte da zero, così lo standard viene collaudato su verde. **Debiti accumulati sul 0645** (non bloccanti, da sanare in futuro retrofit):
    - Rinomina cartella TIA project da `0645_pompe_term/` a `TIA_VCI/` (decisione 2026-04-23, naming astratto cross-commessa).
    - Introduzione cartelle `Contesto/` (con sottocartelle capitolato/schemi/datasheet/manuali) e `_StandardVBA/` (snapshot documentazione, §2b).
    - Spostamento file sciolti alla radice (xlsx, pdf Delta, csv Modbus) dentro `Contesto/datasheet/` o `InterazioniCliente/`.
    - Transizione marker codice `#pat → #paj` per i file già adattati al 0645 (il 0645 è oggi sorgente del modello, vive come se fosse template; serve passaggio di ripulitura quando il modello sarà estratto).
    - Eventuale rinomina `node-red/` se si decide di spostarla dentro `TIA_VCI/`.
5. **Sistema di condivisione documentazione standard**: oggi `_StandardVBA/` è copia manuale. Valutare submodule git, pacchetto npm-like, libreria condivisa, o altro meccanismo. Prerequisito: la documentazione standard sia sufficientemente stabile da giustificare la distribuzione.
6. **Skill `/sync-standard-vba`**: automatizzare la copia `elucubrIAmo/plc/*.md` → `_StandardVBA/` di un progetto, con aggiornamento automatico di `_VERSIONE.md`. Da scrivere quando avremo più progetti attivi.
