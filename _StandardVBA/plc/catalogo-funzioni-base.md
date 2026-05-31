# Catalogo funzioni base — libreria standard PLC VBA

**Stato**: bozza 1, aprile 2026. Estratto da `0645_BancoPompe_plc` in fase di "conversione al modello".
**Scopo**: definire le funzioni e function blocks base/pattern che vivranno nel template VBA, con signature, dipendenze e ciclo di vita marker.

**Convenzioni**:
- Organizzato per **cartella numerata** di `Program blocks/` (ruolo).
- **Marker scope** sui filename: `(nessuno)` = base secca, `#pat` = pattern (solo modello), `#paj` = ex-pattern adattato (solo progetti), `#prj` = project-specific (escluso da questo catalogo).
- **Solo base e pattern**: le funzioni `#prj` (progetto-specifiche) non sono nel catalogo.
- Per ogni voce: signature, scopo, dipendenze classificate (**infrastruttura standard**, **UDT**, **DB locali**, **system tag**).

**Correlati**:
- [convenzioni-progetto-plc.md](convenzioni-progetto-plc.md) §3 — marker scope codice + infrastruttura standard VBA
- [catalogo-udt-base.md](catalogo-udt-base.md) — UDT referenziati nelle signature
- *catalogo-db-templates.md* (step 3) — DB di dati standalone (`dbB`, `dbAIs`, `dbA`, `dbComm`, `dbAiRd`, `dbAOWr`)

---

## Sequenza Main scan cycle (template VBA)

Il `Main` di un progetto PLC VBA segue una sequenza standard di chiamate per ciclo. Estratto dal 0645 con marker rinormalizzati al modello:

```scl
ORGANIZATION_BLOCK "Main"
BEGIN
   "fcDiRd#pat"(i := "dbA".i);              // 1. ingressi digitali con forzature
   "fcPc2s7#pat"();                          // 2. PC→PLC: comandi, setpoint, modo operativo
   "fcGesPort#pat"(ph := "dbB".glob.gate.ph);// 3. gate banco (pattern banco-specifico)
   "fcAiRd#pat"();                           // 4. analogiche IN + riparametrazione + check SP
   "fcS72pc#pat"();                          // 5. PLC→PC: stati, allarmi, analogiche
   "fcAoWr#pat"();                           // 6. analogiche OUT con riparametrazione
   
   // Reset hardware per circuiti inattivi
   IF <circuito olio non in manuale> THEN "fcHwRst#pat"(circuito := "dbB".cOlio); END_IF;
   IF <circuito glic non in manuale> THEN "fcHwRst#pat"(circuito := "dbB".cGlic); END_IF;
   
   // Logica circuito attivo (manuale/automatico)
   <chiamate fc<Manuale|Auto>#pat e gestori specifici banco>
   
   // Termoregolazione circuito attivo
   "fcGTR#pat"(circ := <circuito attivo>);
   
   "fcStepCtrl#pat"();                       // 7. ciclo passi prova
   "fcLogiGen#pat"();                        // 8. logica generale (a-specifica)
   
   "fcAIs#pat"();                            // 9. registrazione allarmi (input → b.input)
   "f_AllSeq"();                             // 10. SM allarmi (suono, lampada, mem)
   
   "fcSafeChk#pat"();                        // 11. check sicurezza trasversali
   
   "fcDoWr#pat"(q := "dbA".q);               // 12. uscite digitali (con forzature/interlock)
   
   "fcMB_init#pat"();                        // 13. init Modbus master (one-shot)
   "fcMBcycle#pat"();                        // 14. ciclo Modbus master (drives)
   "fcGMbSl#pat"();                          // 15. server Modbus slave (per BE/FE)
END_ORGANIZATION_BLOCK
```

**Lettura**: lettura ingressi → traduzione comandi PC → logica applicativa → traduzione stati per PC → registrazione/elaborazione allarmi → safety → scrittura uscite → comunicazione con drive esterni. **Variazione tipica per progetto**: passi 3, 7, 8, 9 (registrazione allarmi) e tutta la logica circuito attivo (passo "logica circuito"). I passi 1, 2, 4, 5, 6, 10, 11, 12 restano sostanzialmente invariati.

---

## `10-Sistema/`

OB di sistema TIA standard. Numerati per ruolo.

| Blocco | Tipo | Marker | Scopo |
|---|---|---|---|
| `Main` | OB1 | (nessuno) | Orchestratore ciclo scan principale. **Il corpo è banco-specifico** ma la **sequenza-base** è quella documentata sopra. Nel template è scheletro con commenti ai punti di personalizzazione. |
| `Startup` | OB100 | (nessuno) | Init al power-on PLC. Tipicamente vuoto o con flag di prima accensione. |
| `Cyc10ms` / `Cyc100ms` / `Cyc250` / `Cyc500` | OB cyclic | (nessuno) | Hook ciclici per logiche temporizzate. `Cyc100ms` ospita tipicamente la chiamata a `fcRegTimedCall#pat` (cascata PID). |
| `ErrorMgt/*` | OB error handlers | (nessuno) | Standard Siemens (DiagnosticErrorInterrupt, IO_AccessError, ProgrammingError, PullOrPlugOfModules, RackOrStationFailure). Sotto-cartella esplicita per origine vendor. |

---

## `20-IO/`

Lettura ingressi e scrittura uscite (digitali e analogiche) con forzature.

### `fcDiRd#pat` — lettura digitali con forzature

```scl
FUNCTION "fcDiRd#pat" : Void
   VAR_IN_OUT
      i : t_di;
   END_VAR
```

**Scopo**: legge memory areas fisiche (`di0`, `di2`, `di4`, `di6` Word raw) con `SWAP` per byte-order, applica MUX di forzatura (`dbCommOtt.diF[].fs/fv`), popola `i.i[]`. Sostituisce gli ingressi reali con i valori forzati quando `fs=1`. Setta anche `dbAIs.ack`/`dbAIs.reset` da bit comando + pulsante hardware.

**Dipendenze**: infrastruttura standard (`dbA`, `dbAIs`, `dbB`), `dbComm`/`dbCommOtt` (DB locali). Memory areas Siemens (`di0`/`di2`/`di4`/`di6`).

**Cosa adattare nel progetto**: il numero di word DI (4 nel 0645 → 64 canali) varia. Il loop `FOR p := 0 TO 15` resta, si replica/elimina blocchi `#k := N` per ogni word in più/meno.

### `fcDoWr#pat` — scrittura digitali con forzature, interlock e device monitorati

```scl
FUNCTION "fcDoWr#pat" : Void
   VAR_IN_OUT
      q : t_dq;
   END_VAR
```

**Scopo**: applica forzature output da `dbCommOtt.doF[].fs/fv`, chiama `f_gMonEv` per ciascuna valvola monitorata, applica interlock di sicurezza (es. riscaldatore consentito solo con pompa OK + portata + T sotto soglia), chiama `f_gDev` per ogni device di uscita scrivendo su memory area Siemens.

**Dipendenze**: infrastruttura standard, `dbB` (per le valvole monitorate `mEv` e flag termoreg `auxQPotenza`/`auxQModulante`), tag fisici Siemens tipo `KMuPOMPOLIO` etc.

**Cosa adattare**: la lista delle valvole monitorate (`f_gMonEv` calls) e la lista dei `f_gDev` sono banco-specifiche. Gli interlock sicurezza riscaldamento sono pattern (struttura uguale, condizioni leggermente diverse).

### `fcAiRd#pat` — lettura analogiche con riparametrazione

```scl
FUNCTION "fcAiRd#pat" : Void
```

**Scopo**: copia memory areas raw nei canali `dbAiRd.anCh[].valPts`, itera `f_RiparaAi` per ogni canale (ripara `valPts → valEng` con `t_anChCfg`), popola `dbB.<area>.pv.*` con i valori finali, chiama `f_AiSpCk` per ogni setpoint con qualità+sicurezza.

**Dipendenze**: infrastruttura standard, `dbAiRd`, `dbCommOtt`, `dbB` (per pv banco).

**Cosa adattare**: la mappa "memory area Siemens → canale `dbAiRd`" è banco-specifica. Il loop generale `f_RiparaAi`/`f_AiSpCk` resta.

### `fcAoWr#pat` — scrittura analogiche con riparametrazione

```scl
FUNCTION "fcAoWr#pat" : Void
```

**Scopo**: copia valori engineering da `dbB.<circuito>.cPres.outEng` in `dbAOWr.aoCh[].valEng`, chiama `f_riparaAo` per convertire eng → pts, scrive su memory area fisica con clamping.

**Dipendenze**: `dbAOWr`, `dbCommOtt` (tarature), `dbB`, tag fisici tipo `CPaoRESSOLIO`.

**Cosa adattare**: i canali AO del banco e la mappa eng→pts. Pattern di chiamata a `f_riparaAo` invariato.

### `fcHwRst#pat` — reset hardware selettivo per circuito

```scl
FUNCTION "fcHwRst#pat" : Void
   VAR_IN_OUT
      circuito : t_dbBCircuito;
   END_VAR
   VAR_TEMP
      mask : t_rstSkipMask#pat;
      resetPhSlave : Bool;
      resetOutputs : Bool;
   END_VAR
```

**Scopo**: reset selettivo del circuito non attivo (azzera comandi valvole monitorate, flag pompe, comandi termoregolazione) rispettando una maschera `t_rstSkipMask#pat` che permette di escludere singoli elementi.

**Dipendenze**: solo l'argomento + `t_rstSkipMask#pat`. Buona astrazione: la funzione opera **solo** sull'argomento `circuito`.

---

## `30-Comunicazione/`

PC↔PLC, Modbus master verso drive, Modbus slave verso BE/FE.

### `fcPc2s7#pat` — decodifica comandi PC→PLC

```scl
FUNCTION "fcPc2s7#pat" : Void
```

**Scopo**: arbitraggio HMI vs WEB sulla scrittura comandi (Word di 32 bit `pc2plcCmd[0..1]` → struct `t_cmd2plc#pat`), copia setpoint, gestione modi operativi (`benchMode` da bit comando), copia `cMan` da bit individuali in struct `cmdSvc`.

**Dipendenze**: `dbB`, `dbComm`, `dbCommOtt`, `dbAIs`, costanti `cfgBmManOil`/`cfgBmManGlic`/`cfgBmManTerm`/`cfgBmAutOil`/etc., `cfgTestOil`/`cfgTestGlic`/`cfgTestTerm`.

**Cosa adattare**: la mappa bit-comando → bit-`cmdSvc` (`cMan.olio := req2plc.ol10io`, ecc.) varia se i nomi semantici dei bit di `t_cmd2plc#pat` sono diversi. La logica di arbitraggio HMI vs WEB è invariante.

**Pattern da preservare**: la funzione esplicita le 32 conversioni bit↔Word in entrambe le direzioni, scegliendo via "diff" tra ottimizzato e non se la sorgente del cambio è HMI o WEB. Non sintetizzabile, va manualmente replicato per ogni bit.

### `fcS72pc#pat` — encoding stati PLC→PC

```scl
FUNCTION "fcS72pc#pat" : Void
```

**Scopo**: copia `dbAIs.a[].b.mem` in word allarmi `dbCommOtt.alarms[]`, popola feedback PLC→PC (`req2pc.*`: gate, fase corrente, inRange, allarme attivo, banco pronto), encoding bit→Word (`plc2pcCmd[]`), copia analogiche e digitali virtuali in formato leggibile da PC.

**Dipendenze**: `dbB`, `dbA`, `dbAIs`, `dbAiRd`, `dbComm`, `dbCommOtt`, costanti `cfgNAlarmW`, `cfgNAi`, `cfgNDi`, `cfgNDiW`, `cfgNDo`, `cfgNDoW`.

### `fcUpdSlow#pat` — update lento di parametri

```scl
FUNCTION "fcUpdSlow#pat" : Void
```

**Scopo**: tipicamente chiamato in `Cyc500` o simile, copia setpoint da `dbB.glob.sp.tOut.*` (timeout/watchdog) in `dbAIs.a[].insTmcSp` per gli allarmi che usano ritardi configurabili da PC.

**Dipendenze**: `dbAIs`, `dbB`.

### `fcGMbSl#pat` — server Modbus slave (wrapper su MB_SERVER_DB Siemens)

```scl
FUNCTION "fcGMbSl#pat" : Void
```

**Scopo**: chiama `MB_SERVER_DB` (FB Siemens) con i parametri standard VBA (`dbA.pcCon`, `dbA.pc*` per status/error, `dbComm` come holding register).

**Dipendenze**: `MB_SERVER_DB` (Siemens, in `Esterni`), `dbA`, `dbComm`.

### `fcMB_init#pat` — init Modbus master (one-shot)

```scl
FUNCTION "fcMB_init#pat" : Void
```

**Scopo**: configura `dbComm_Data.Param.*` (Baud, Parity, Resp_to, ecc.) e `dbComm_Data.Master_comm[N]` (slave addr, mode, data addr, data len) per ciascun drive esterno. Esegue una sola volta tramite flag `dbA.mbInit`.

**Cosa adattare**: lista degli slave Modbus (drive presenti nel banco) — tipicamente 1-3 drive con 2-3 transazioni cad.

### `fcMBcycle#pat` — ciclo Modbus master (lettura/scrittura drive)

```scl
FUNCTION "fcMBcycle#pat" : Void
```

**Scopo**: gestisce condizionalmente lo slave attivo (azzera `MB_ADDR` se drive spento per evitare timeout), chiama `dbMaster_Modbus_DB` per ogni transazione, decodifica buffer raw nei campi di `dbB.<area>.drv.naif/pts/eng`, chiama `f_deltaGetStatus` per decodifica status, `f_deltaSetCmds` per encoding comandi.

**Cosa adattare**: stessa lista degli slave di `fcMB_init`, mappatura buffer→campi specifica per ciascun drive vendor.

### `Master_Modbus.scl` (codice Siemens, no marker)

Sotto-cartella `MB master/`, codice generato da Siemens dall'esempio MB master. **Non si tocca**, vive nella cartella vendor.

---

## `40-Regolazione/`

PID, PWM, termoreg, gestione motori, valvole proporzionali, allarmi, regimazione, safety.

### `fcAIs#pat` — registrazione allarmi (input → struct alarm)

```scl
FUNCTION "fcAIs#pat" : Void
```

**Scopo**: per ciascun allarme configurato del banco, copia bit input fisico nel campo `b.input` di `dbAIs.a[N]` e propaga `b.mem` a flag visibile (es. `dbAIs.SNaDROKTRAS := dbAIs.a[1].b.mem`). Poi gestisce muting allarmi quando `SIaTUTTO_OK_=FALSE` (sistema non pronto, tutti gli allarmi mascherati).

**Dipendenze**: `dbAIs`, `dbA` (input fisici), `dbB` (per allarmi calcolati su valori processo).

**Cosa adattare**: la lista degli allarmi (40 nel 0645) è banco-specifica. Pattern: 2 righe per allarme (`flag := mem`, `b.input := <espressione>`).

### `fcRegTimedCall#pat` — chiamate temporizzate PID

```scl
FUNCTION "fcRegTimedCall#pat" : Void
```

**Scopo**: chiama `f_pid` per ogni PID del banco. Tipicamente invocato da `Cyc100ms` o simile, per garantire `DeT` costante.

**Dipendenze**: `dbB`.

**Cosa adattare**: l'elenco dei PID (termoreg olio/glic risc/raff, valvole proporzionali pres/pos, booster, trascinamento UUT nel 0645).

### `fcSafeChk#pat` — controlli sicurezza trasversali

```scl
FUNCTION "fcSafeChk#pat" : Void
```

**Scopo**: forza spegnimento di trascinamento, booster, termoreg, pompe quando: (a) nessun circuito attivo, (b) allarme generale `SIaTUTTO_OK_` attivo, (c) interblocco portellone+motore non rispettato.

**Dipendenze**: `dbB`, `dbA`, `dbAIs`.

**Cosa adattare**: la lista degli interblocchi e i campi `dbB` da forzare. La logica generale (3 condizioni, override gateOvride) è invariante.

### `fcGTR#pat` — gestione termoregolazione (heat + cool con SOC)

```scl
FUNCTION "fcGTR#pat" : Void
   VAR_IN_OUT
      circ : t_dbBCircuito;
   END_VAR
```

**Scopo**: state machine separate per riscaldamento (`phH`) e raffreddamento (`phC`) di un circuito. Cascata PID + PWM + check SOnda in Corto. Stati: `notRdy → waitCmd → rdy → regima → runNormal → riscShutOff → notRdy` (heat) e simile per cool. Gestisce fault e recovery.

**Dipendenze**: `t_dbBCircuito` IN_OUT, `dbB.glob.cmd.cWrk` (comandi standard), chiama `f_pttTm`.

**Da rifattorizzare** (tu hai messo `#pat` come segnaposto): per essere veramente astratta dovrebbe ricevere `cmdRisc`/`cmdRaff` come `Bool` IN, eliminando l'accesso diretto a `dbB.glob.cmd.cWrk.risc/raff`.

---

## `50-Funzioni/`

Helper riusabili (`f_*`) e funzioni applicative (alcune ancora `f_` per convenzione storica anche se non perfettamente astratte).

### Astratte pure (no marker, no dipendenze esterne)

| Funzione | Signature | Scopo |
|---|---|---|
| `f_pid` | `pid : t_PIDstr` IN_OUT | PID standard ISA con derivata filtrata su PV, anti-windup su BIAS, normalizzazione interna 0-100% |
| `f_pwm` | `a : t_pwm` IN_OUT | PWM su digital output con periodo configurabile, embeddata `t_ph` per stato ON/OFF |
| `f_AiSpCk` | `value : Real` IN, `sp : t_stepSp` IN_OUT | Aggiorna bit di status (`hs4alm`/`hq2alm`/`lq1alm`/`ls3alm`/`in0range`) di un setpoint complesso con qualità+sicurezza |
| `f_PickupSpeedUpdate` | `Ts_s : LReal` IN, `cnt : t_hsCnt` IN_OUT | Counter veloce → calcolo Hz/rpm/rad_s con buffer circolare per averaging |
| `f_RiparaAi` | `aiCfg : t_anChCfg` IN, `ai : t_aiCh`/`aiCom : t_aiMbCh` IN_OUT | Riparametrazione raw→eng con i Gefran (Tmist/Tmast/ItMist/ItMast), media mobile, fault overrange/underrange |
| `f_riparaAo` | `aoCfg : t_anChCfgAo` IN, `ao : t_aoCh` IN_OUT | Riparametrazione eng→raw moderna (EngMin/Max/Idle, PtMin/Max/Idle), clamp e fault |
| `f_pttRst` | `ph : t_ph` IN_OUT | Reset di un `t_ph` (timer + storico) |
| `f_deltaGetStatus` | `value : Word` IN, `drive : t_DeltaDrv` IN_OUT | Decodifica word di stato Delta drive in bit semantici |
| `f_deltaSetCmds` | `drive : t_DeltaDrv` IN_OUT | Encoding bit semantici → word comando Delta drive |

### Astratte con dipendenze su infrastruttura standard VBA

| Funzione | Signature | Dipendenze | Note |
|---|---|---|---|
| `f_pttTm` | `ph : t_ph` IN_OUT | `dbA.TMcAlive` | Aggiorna `ph.TMcElap` su clock di sistema, scala storico al cambio fase. **Pilastro** delle state machine — chiamata da quasi ogni `fc/f_` con `t_ph`. |
| `f_gDev` | `dev : t_dev` IN_OUT, `q : Bool` OUT | `Clock_1Hz` | Applica forzature/inibizioni su `t_dev`, mantiene `tmsRun` (secondi attivazione) e `cntStart` (contatore avvii) |
| `f_gMonEv` | `resetFault : Bool` IN, `mEv : t_monEv` IN_OUT | `dbA.i.i[]`, `dbA.q.dev[].i` | State machine di monitoraggio dispositivo con feedback fcOff/fcOn (valvola, contattore). Indici dei canali sono dentro `t_monEv` (parametrizzati). |
| `f_AllSeq` | (nessun argomento) | `dbAIs.*`, `cfgNAlarm`, chiama `f_pttTm` | State machine allarmi: gestisce sequenza `inattivo → attesa insensibilità → sonoro → silenzioso → reset` per ogni `dbAIs.a[N]`, popola flag globali `dbAIs.flSpiaOn`/`flSonoro`. **Itera tramite indice su array dimensionato da costante** — pattern astratto via parametrizzazione strutturale. |

### Pattern (#pat)

| Funzione | Signature | Scopo | Cosa adattare |
|---|---|---|---|
| `f_soc#pat` | `ch : t_soc` IN_OUT | SOnda in Corto: condizione di risposta attesa entro finestra temporale | Attualmente accede a `dbA.q.dev[KMuRISCGLIC]` e `[YVuRAFFTRAS_2040]` — refactor pendente per ricevere `running : Bool` come IN. |

### `f_CfGEvProp` — borderline, lasciata senza marker

```scl
FUNCTION "f_CfGEvProp" : Void
   VAR_IN_OUT
      cPres : t_evProp;
   END_VAR
```

**Scopo**: state machine controllo valvola proporzionale a cascata pres→pos, con ricerca punto lavoro, stabilizzazione, mantenimento posizione, fault.

**Status**: tecnicamente accetta solo un argomento di tipo `t_evProp` (definito in `Progetto/Macro/`), ma accede internamente a `dbB.glob.cmd.req2plc.co16ntropressa` e `dbAIs.ack`. Lasciata senza marker per scelta di Roberto: il pattern è considerato standard VBA per gestione valvole proporzionali, l'accesso a `co16ntropressa` rientra nelle **convenzioni di interfaccia bit semantica** (vedi convenzioni §3 — *Infrastruttura standard VBA*).

**Da rifattorizzare nel medio termine**: ricevere `cmdEnable : Bool` IN al posto dell'accesso diretto a `dbB.glob.cmd.req2plc.co16ntropressa` la renderebbe pienamente astratta.

---

## `60-Mapping/`

Conversione fisico↔virtuale dei segnali (mapping di astrazione).

Nel 0645 le due funzioni (`fcMapFis2virt#prj`, `fcMapVirt2fis#prj`) sono **project-specific**: la mappatura dipende dalla fisica del banco. Pattern non emerso ancora come riusabile — **per ora niente in catalogo base**, candidata a futura promozione se più progetti adottano lo schema.

---

## `80-Init/`

### `fcInit#pat` — inizializzazione progetto (default + configurazione)

```scl
FUNCTION "fcInit#pat" : Void
```

**Scopo**: chiamato una volta al power-on (verifica `dbCommOtt.aiTara[0].ItMast = 0`). Inizializza:
- Tarature default per tutti i canali AI (`Tmist/Tmast/ItMist/ItMast/TmiFlt/TmaFlt/kNew`).
- Configurazione di tutte le valvole monitorate del banco (`fcOffNdx/fcOnNdx/qNdx/graceTime_c` per ogni `dbB.<circ>.mEv.YVu*`).
- Configurazione PID valvole proporzionali (P/I/D/DeT/Range).
- Configurazione tarature uscite analogiche (`dbCommOtt.aoTara[].EngMin/Max/Idle/PtMin/Max/Idle`).

**Dipendenze**: `dbCommOtt`, `dbB`, `dbA` (per indici mnemonici `id`/`qd`).

**Cosa adattare**: tutta la configurazione è banco-specifica per istanza, struttura del file (sezioni REGION) è invariante.

---

## `90-Archivio/`

Blocchi obsoleti tenuti per riferimento storico. Marker `#prj` per quelli nati in un singolo progetto e poi abbandonati. **Non vanno nel catalogo base.**

Esempio nel 0645: `fcSmMacro#prj.scl` (l'architettura `Macro state machine + slave SM` proposta dall'AI e poi rollbackata — vedi anti-pattern-osservati.md quando lo scriveremo).

---

## Pattern strutturali ricorrenti

### Pattern "loop su array dimensionato"

Tipico per: scrittura/lettura I/O, encoding/decoding word di comandi, registrazione allarmi.

```scl
FOR #n := 0 TO "cfgN<categoria>" DO
   <azione su elemento [#n]>
END_FOR;
```

Le costanti `cfgN*` sono **infrastruttura standard VBA** — la funzione resta astratta.

### Pattern "state machine con t_ph"

Standard per qualunque comportamento temporale.

```scl
"f_pttTm"(#oggetto.ph);    // aggiorna TMcElap, gestisce storico

CASE #oggetto.ph.ptr OF
   #stato0:
      IF <transizione> THEN #oggetto.ph.ptr := #stato1; END_IF;
   #stato1:
      ...
END_CASE;
```

Costanti di stato dichiarate in `VAR CONSTANT` con prefisso semantico (es. `pmeNotRdy`, `pmeRdy`, `pmeOff`, `pmeOn`, `pmeFault` per `f_gMonEv`).

### Pattern "encoding/decoding bit ↔ Word"

Tipico per comunicazione PC↔PLC (Modbus, S7TCP).

```scl
// Encoding bit → Word
#word := 0;
IF #struct.bit0name THEN #word := #word OR 16#0001; END_IF;
IF #struct.bit1name THEN #word := #word OR 16#0002; END_IF;
...

// Decoding Word → bit
#struct.bit0name := (#word AND 16#0001) <> 0;
#struct.bit1name := (#word AND 16#0002) <> 0;
...
```

Sintetizzabile con SHL/SHR ma il pattern esplicito è più leggibile e testabile.

### Pattern "decomposizione device esterno (5+1)"

Per qualunque drive/strumento via fieldbus: separare `ph + eng + pts + naif + sp + onLine` (vedi `t_DeltaDrv` in catalogo UDT). Le funzioni gemelle `f_<vendor>GetStatus` (decodifica word → bit semantici) e `f_<vendor>SetCmds` (encoding bit → word comando) sono il template per nuovi vendor.

---

## Questioni aperte

1. **`f_soc`**: refactor pendente per ricevere `running : Bool` come IN ed eliminare dipendenza da tag fisici. Per ora `#pat`.
2. **`fcGTR#pat`**: refactor per ricevere `cmdRisc`/`cmdRaff` come Bool IN, oggi accede direttamente a `dbB.glob.cmd.cWrk.*`.
3. **`f_CfGEvProp`**: borderline — la sua qualità "astratta" dipende dal principio "bit semantici di pattern bitfield standard sono interfaccia VBA". Se accettato, resta `f_`. Se irrigidiamo il principio, va `#pat`.
4. **`60-Mapping/`** vuoto nel catalogo base — emerge un pattern riusabile dopo 2-3 progetti?
5. **Word 0 di `t_cmd2plc#pat`** — i 16 bit (allarmi/marcia/stop/conferma/cancel) dovrebbero essere realmente standard VBA. Definire la lista canonica (oggi nel 0645 c'è ma non è formalizzata come "standard VBA Word 0").
6. **Standardizzazione bit `t_cmd2pc#pat`** — analogamente, i 16 bit di feedback (gate, banco pronto, fase, inRange, allarme) potrebbero essere standard VBA. Da formalizzare.
