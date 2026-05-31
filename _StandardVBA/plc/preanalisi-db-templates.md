# Pre-analisi DB templates — preparazione step 3

**Stato**: bozza preparatoria, 19 aprile 2026.
**Scopo**: leggere e classificare le DB del 0645 in vista del catalogo `catalogo-db-templates.md`. Identificare pattern strutturali, scelte di progettazione, marker da assegnare. Non è ancora il catalogo finale — è materiale per discussione.

**Nota tecnica**: i file `.db` del 0645 sono **testo UTF-8 con BOM** (non binari come sembrava dal Read tool). Letti tramite `cat`. Ogni DB è composto da una sezione `VAR ... END_VAR` (dichiarazione struttura) + una sezione `BEGIN ... END_DATA_BLOCK` (valori di init).

## Inventario DB in 0645

| DB | Posizione | Dim | Tipo | Marker proposto |
|---|---|---|---|---|
| `dbA` | `70-DB/` | 3 KB | Wrapper IO + clock + TCP server | **`#pat`** (struttura ricorrente, indici banco-specifici) |
| `dbAIs` | `70-DB/` | 69 KB | Allarmi banco | **`#pat`** (struttura ricorrente, lista allarmi banco-specifica) |
| `dbAiRd` | `70-DB/` | 3 KB | Lettura analogiche raw | **`#pat`** (lista canali banco-specifica) |
| `dbAOWr` | `70-DB/` | 0.5 KB | Scrittura analogiche | **`#pat`** o `#prj` (per ora solo 2 canali, banco-specifici) |
| `dbB` | `70-DB/` | 105 KB | Dati di banco (cuore) | **`#pat`** (struttura ricorrente, contenuto totalmente banco-specifico) |
| `dbComm` | `30-Comunicazione/` | 23 KB | Interfaccia BE/FE non-ottimizzata (layout protocollo) | **`#pat`** |
| `dbCommOtt` | `30-Comunicazione/` | 14 KB | Interfaccia BE/FE ottimizzata (copia interna) | **`#pat`** |
| `dbMbTcpSI` | `30-Comunicazione/` | 0.5 KB | Config TCP secondaria (?) | da chiarire |
| **DB Esterni Siemens** | `30-Comunicazione/MB master/`, `System blocks/` | varie | `dbMaster_Modbus_DB`, `dbOutput_Data`, `dbComm_Data`, `MB_SERVER_DB` | **fuori catalogo** (esterni vendor) |

## Pattern strutturali emersi

### Pattern 1 — Wrapper minimo + delega a UDT (`dbB`)

```scl
DATA_BLOCK "dbB"
   VAR
      glob : t_dbBGlob;        // dati globali
      cOlio : t_dbBCircuito;   // circuito olio
      cGlic : t_dbBCircuito;   // circuito glicole
   END_VAR
   BEGIN
     // ~3000 righe di init dei valori default
   END_DATA_BLOCK
```

**Dichiarazione** è 3 righe: tutta la complessità è negli UDT in `Progetto/Macro/`. Tutti i 105 KB del file sono **valori di default** nel BEGIN. Il pattern: DB minimale + delega + init estensiva.

**Implicazione per il modello**: `dbB` template avrà la stessa struttura di 3 membri ma init **vuoto** o con i soli valori critici (limiti sicurezza, default ragionevoli). I valori specifici del banco si popolano in fase di adattamento.

### Pattern 2 — Tre rappresentazioni per ogni elemento (`dbAIs`)

Per ciascun allarme convivono **3 forme** nello stesso DB:

```scl
"SIaTUTTO_OK_" : Bool;     // 0  - flag semantico (la "a" sta per "alarm")
SInTUTTO_OK_ : Int;         // 0  - costante indice (la "n" sta per "number")
a[0] : t_alStr;              // record completo nell'array (in cfgNAlarm slots)
```

E nel codice (es. `fcAIs#pat`):
```scl
"dbAIs"."SIaTUTTO_OK_" := "dbAIs".a["dbAIs"."SInTUTTO_OK_"].b.mem;
```

**Vantaggio**: il codice può accedere all'allarme **per nome** (leggibilità) **o per indice** (parametrizzazione). La triade è ridondante ma esplicita la convenzione VBA.

**Implicazione per il modello**: il pattern triade va documentato come **convenzione VBA per allarmi**. Nuovi banchi seguono lo stesso schema (modificando solo i nomi semantici e il numero di allarmi via `cfgNAlarm`).

**Nota**: nel 0645 ci sono `Static_a40..47` e `Static_n40..47` come **placeholder per espansione futura** (analoga ai `flN` nei pattern bitfield). Confermare se è convenzione standard.

### Pattern 3 — Mnemonico → indice (lookup table) (`dbA`, `dbAiRd`, `dbAOWr`)

Pattern usato per IO digitali, analogiche e indici di canale:

```scl
DATA_BLOCK "dbA"
   VAR
      i : t_di;                  // array di Bool dimensionato da cfgNDi
      id : t_di_def#pat;         // struct con un nome per ogni canale
      q : t_dq;
      qd : t_dq_def#pat;
   END_VAR
   BEGIN
      id.SNeMODSICOK := 16#00;   // ciascun nome → indice fisso
      id.SNeDROKTRAS := 16#01;
      ...
      qd.SNuCONSTRAS := 16#00;
      ...
   END_DATA_BLOCK
```

**Nel codice** si accede così:
```scl
"dbA".i.i["dbA".id.SNeMODSICOK]   // lettura DI mnemonica
"dbA".q.dev["dbA".qd.YVuRABBOLIO_670].i := ...  // scrittura DQ mnemonica
```

**Eleganza**: il codice è auto-documentato (`id.SNeMODSICOK` invece di `i[0]`), il refactor degli indici è centralizzato (basta cambiare il valore di init), la dimensione dell'array è parametrica via `cfgNDi`/`cfgNDo`.

### Pattern 4 — DB doppio: layout protocollo + copia ottimizzata (`dbComm` + `dbCommOtt`)

I due DB hanno **struttura identica** ma differiscono per:
- `dbComm` ha `S7_Optimized_Access := 'FALSE'` → **layout fisso in memoria** → mappato direttamente sui Modbus holding register letti dal BE/FE
- `dbCommOtt` ha `S7_Optimized_Access := 'TRUE'` → **layout ottimizzato dal compilatore** → usato dal codice per accessi efficienti

`fcPc2s7#pat` e `fcS72pc#pat` fanno la **copia bidirezionale** dei due DB ad ogni ciclo. Il `dbComm` è un "package" stabile per il protocollo, il `dbCommOtt` è la "vista interna" del codice.

**Implicazione per il modello**: la coppia `dbComm + dbCommOtt` è uno **standard VBA** per qualunque interfaccia con protocollo a layout fisso. La struttura interna include sezioni standard (`diF`, `doF`, `pc2plcCmd`, `aiTara`, `alarms`, `plc2pcCmd`, ecc.) che dovrebbero essere parte del template.

### Pattern 5 — Spazio "tappo" per espansione layout

```scl
tappo : Array[0..33] of Word;
tappoAllRd : Array[0..99] of Word;
end_of_2plc : Word;
end_of_2pc : Word;
```

In `dbComm` ci sono campi "tappo" che riservano spazio nel layout per future espansioni senza spostare gli offset dei campi successivi. Pattern di progettazione esplicito per protocolli a layout fisso (regola operativa: *"MAI correggere offset 1300/HR650 di dbComm zona lettura — se manSp cresce, regola tappoAllRd per compensare"*, già nota nel `_todo.md` del 0645).

**Da formalizzare** nel catalogo: i tappo sono parte essenziale del pattern, non un dettaglio implementativo.

## Distinzione: dichiarazione vs init nei DB

Una decisione importante: i DB del 0645 hanno spesso **migliaia di righe di init** nel `BEGIN`. Strutturalmente ridondante ma operativamente utile (default verificati al primo avvio).

**Domande per il modello**:

1. **Quanto init del modello mantenere**? I default standard (es. `aiTara[].kNew := 0.01`) sono sensati ovunque. I default specifici (es. `aiTara[8].Tmist := 6042.0` da taratura banco) sono solo del 0645.

2. **Strategia init**: il `fcInit#pat` carica i default a runtime se il DB è vuoto (verifica `if dbCommOtt.aiTara[0].ItMast = 0`). **Questo riduce la necessità di init nel DB**. Pattern coerente per il modello: DB con init **minimo o assente**, fcInit#pat che inizializza al primo avvio.

3. **Init nel BEGIN vs init in fcInit**: la prima è più rapida (compila), la seconda più flessibile (può essere ri-eseguita per reset). Servirebbe una regola chiara nel modello.

## Proposta struttura `catalogo-db-templates.md`

Quando lo scriveremo, organizzazione per ruolo:

1. **Wrapper IO + sistema** — `dbA` (clock, vettori IO con mnemonici, TCP config)
2. **Lettura analogiche raw** — `dbAiRd` (canali AI con mnemonici)
3. **Scrittura analogiche** — `dbAOWr` (canali AO)
4. **Allarmi** — `dbAIs` (triade flag/indice/record + flag globali)
5. **Dati di banco** — `dbB` (wrapper minimo + delega a `Progetto/Macro/`)
6. **Comunicazione PC↔PLC** — `dbComm` + `dbCommOtt` (coppia layout/ottimizzato + tappo)
7. **Pattern strutturali** — sezione separata che documenta i 5 pattern emersi (wrapper minimo+delega, triade, mnemonico→indice, doppio DB, tappo per espansione)

Per ogni DB: signature (membri principali, non tutti i campi), pattern utilizzato, dipendenze UDT, "cosa adattare" nel passaggio da modello a progetto.

## Domande aperte da risolvere prima del catalogo finale

1. **Init nel DB o in fcInit#pat**? Decisione architetturale globale.
2. **`Static_a40..47` placeholder allarmi** è convenzione standard (analoga a `flN` nei bitfield)? Da formalizzare.
3. **`dbMbTcpSI`** (490 byte): è una seconda connessione TCP separata, oppure residuo? Da chiarire.
4. **`dbAOWr`** è davvero pattern (riusabile) o oggi è troppo banco-specifico (solo 2 canali olio/glicole)? Forse `#pat` con array dimensionato da costante.
5. **Marker `#pat` su DB**: come per UDT/codice, segna "scheletro da adattare". Per i DB l'adattamento riguarda principalmente i **valori di init**. Confermare la semantica per i DB.
6. **Nomi delle istanze nei progetti**: il dbB resta `dbB` o assume nomi banco-specifici? Mia preferenza: nome stabile (`dbB` resta `dbB` in tutti i progetti, contenuto cambia). Coerente col principio "infrastruttura standard VBA" — `dbB` come *nome* è infrastruttura, *contenuto* è banco.

## Cosa farò domani (quando dai OK)

1. Lettura completa di `dbB` (sezioni rimanenti, capire le dimensioni di `t_dbBGlob` e `t_dbBCircuito`).
2. Lettura completa di `dbAIs` (sezione init).
3. Lettura completa di `dbComm`/`dbCommOtt` (sezione finale).
4. Stesura `catalogo-db-templates.md` con pattern formalizzati.
5. Aggiornamento sezione §3 di convenzioni con regola di marker DB e pattern coppia layout/ottimizzato.
