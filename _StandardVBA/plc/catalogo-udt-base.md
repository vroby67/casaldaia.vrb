# Catalogo UDT base — libreria standard PLC VBA

**Stato**: bozza 2, aprile 2026. Estratto da `0645_BancoPompe_plc` in fase di "conversione al modello" (tutti i pattern marcati `#pat`).
**Scopo**: definire il nucleo di UDT che devono essere disponibili **in ogni progetto PLC VBA**, con semantica, uso e ciclo di vita documentati. Sorgente per la futura libreria TIA `VBA Standard`.

**Cicli di vita di un UDT VBA standard:**
- **Modello/template** (futuro `elucubrIAmo/plc/template-progetto/`): contiene gli UDT con `#pat` mantenuto.
- **Progetto adattato**: ogni `#pat` rinominato perde il suffisso e i campi placeholder vengono sostituiti con quelli specifici dell'impianto.
- **`0645_BancoPompe_plc`** è oggi la **fonte** del modello: tutti gli UDT che richiedono adattamento sono marcati `#pat` (anche dove il 0645 stesso li ha già adattati). Quando il modello sarà estratto in template, il 0645 verrà ripulito (suffisso `#pat` rimosso dove l'adattamento è già fatto).

**Organizzazione**: i tipi sono raggruppati per **cartella TIA reale** (`Base/00-fondamentali/`, `Base/10-IO_digitali/`, …). Per ogni voce: cosa è, quando si usa, quando NON, note sul ciclo di vita pattern.

---

## `Base/00-fondamentali/`

UDT atomici sui quali è costruito tutto il resto. Praticamente ogni UDT con stato temporale embedda `t_ph`.

### `t_ph` — phase handler

```scl
STRUCT
   ptr     : Int;                        // puntatore fase corrente
   oldPtr  : Int;                        // puntatore fase precedente (rising edge detection)
   dbg     : Int;                        // sfasamento ptr-oldPtr (debug visuale)
   tmIn    : UDInt;                      // timestamp ingresso fase
   TMcElap : UDInt;                      // tempo trascorso in fase (centesimi)
   hist    { S7_SetPoint := 'False' } : Array[0..9] of t_phHiStep;
   ptrHist : Int;
END_STRUCT
```

**Uso**: instanziato in qualunque struct con state machine (motore, valvola monitorata, segmento ciclo, PID con soft start, PWM, termoregolatore, allarme…).

**Pattern**:
- Cambio fase: `ph.ptr := nuovoStato;` → la funzione di ciclo (`pttTm` o equivalente) aggiorna `oldPtr`, azzera `tmIn`, scala lo storico.
- Timeout in fase: `IF ph.TMcElap >= REAL_TO_INT(durata_s * 100.0) THEN …`
- Debug post-mortem: `hist[]` mostra le ultime 10 fasi attraversate.

### `t_phHiStep` — entry storico fasi

```scl
STRUCT
   phNo   : Int;    // numero fase
   lasted : UDInt;  // durata permanenza (centesimi)
END_STRUCT
```

Solo dentro `t_ph.hist[]`.

### `t_dev` — stato dispositivo digitale gestito

```scl
STRUCT
   i        : Bool;    // input logico (comando da programma)
   iInh     : Bool;    // inibizione
   iFs      : Bool;    // force select (forzatura attiva)
   iFv      : Bool;    // force value (valore forzato)
   q        : Bool;    // output applicato (dopo inh/forzature)
   appQ     : Bool;    // output ciclo precedente (rising edge)
   appCk1s  : Bool;    // check 1s per time-keeping
   tmsRun   : UDInt;   // tempo di funzionamento in secondi (pattern tm[UM]Semantic)
   cntStart : UDInt;   // contatore avvii
END_STRUCT
```

**Uso**: ogni uscita digitale "gestita" è un `t_dev`. Pipeline: comando → inibizione → forzatura → output → diagnostica.

**Punto di forza**: la forzatura è dentro il tipo, non in zona separata. `iFs=1, iFv=V` → `q=V` bypassando `i`.

### `t_anyPtr` — rappresentazione ANY pointer S7

```scl
STRUCT
   s7Code     : Byte := 16#10;
   dataType   : Byte;
   length     : Int;
   dbNumber   : Int;
   memArea    : Byte;
   byteAdrMsb : Byte;
   byteAdrLsb : Word;
END_STRUCT
```

Tecnicismo S7 per pointer generici. **Non aggiungere senza un caso d'uso concreto.**

### `t_cBitInt` — campo bit forzabili (16 bit)

```scl
STRUCT
   fs : Word;   // force select (1 bit per canale = forzato)
   fv : Word;   // force value (1 bit per canale = valore forzato)
END_STRUCT
```

**Uso**: gruppetto di 16 bit con forzatura manuale. `fs.X=1` ⇒ il valore di X diventa `fv.X`.

**Vedi anche** `t_cBitIntX`/`t_cBitInt32` — variante a 32 bit, naming bit con tre opzioni proposte (decisione alla prima occasione d'uso). Per la convenzione naming bit, vedi sezione *Convenzioni emerse → Naming bit con indice hex inline*.

---

## `Base/10-IO_digitali/`

### `t_di` — banco ingressi digitali

```scl
STRUCT
   i : Array[0..cfgNDi] of Bool;
END_STRUCT
```

**Uso**: tutti gli ingressi del banco in un unico array. `cfgNDi` costante utente. Indicizzato con costanti `CPdiXXX`. Popolato da `fcDiRd`.

### `t_dq` — banco uscite digitali

```scl
STRUCT
   debug : Bool;
   dev   : Array[0..cfgNDo] of t_dev;
END_STRUCT
```

**Uso**: tutte le uscite "gestite" del banco. Iterato da `fcDoWr`.

### `t_di_def#pat` — definizione mnemonica canali DI

```scl
STRUCT
   debug         : Bool;
   <NomeCanale1> : Byte := 0;   // descrizione del segnale
   <NomeCanale2> : Byte := 1;   // …
   …
END_STRUCT
```

**Uso**: associa a ogni canale digitale del banco un **nome mnemonico** + **indice fisso** + **commento descrittivo**. Es. nel 0645: `SNeMODSICOK : Byte := 0` (modulo sicurezza riarmato), `YVeVRABOIOP_670 : Byte := 16` (Ev. rabbocco olio aperta), ecc.

**Adattamento**: in ogni progetto si **riscrive** il contenuto con i canali del proprio banco. Lo `#pat` ricorda che la struttura è da popolare ex novo. Convenzione nomi canali: sigla 2-3 char tipologia (`SN`/`YV`/`TE`/`PS`/`LV`/`FC`/…), `e` per ingresso (`q` per uscita?), descrittore semantico, eventuale numero hardware (`_670` ecc.).

### `t_dq_def#pat` — definizione mnemonica canali DQ

Analogo a `t_di_def#pat`, lato uscite.

### `t_hsCnt` — high-speed counter (wrapper su libreria Siemens DI8HS)

```scl
STRUCT
   Ctrl { S7_SetPoint := 'False' } : LPD_typeDI8HSCountControlCh;   // tipo Siemens (Esterni/)
   Fbk                              : LPD_typeDI8HSCountFeedbackCh;  // tipo Siemens
   ppr      { S7_SetPoint := 'True' } : Real;     // pulses per revolution (configurabile)
   lastCount : UDInt;
   delta     : UDInt;
   Hz        : LReal;
   rpm       : LReal;
   rad_s     : LReal;
   NeedGateToggle : Bool;
   cntBuff : Array[0..15] of UDInt;   // buffer circolare per averaging
   ptrBuff : Int;
   deCnt   : Int;
END_STRUCT
```

**Uso**: wrapper sopra la libreria Siemens per scheda DI8HS. Calcola Hz/rpm/rad_s da counter + media mobile su 16 campioni. Si usa quando il banco ha la scheda counter veloce.

**Dipendenza**: i tipi `LPD_type…` vivono in `Esterni/` e non sono sincronizzabili via VCI (protetti).

---

## `Base/20-IO_analogici/`

### `t_aiCh` — canale AI standard

```scl
STRUCT
   valPts   : Real;   // valore raw punti (post-conversione ADC)
   valEng   : Real;   // valore in unità ingegneristiche (post-taratura)
   avgVal   : Real;   // media mobile per stabilizzazione
   done     : Bool;
   fault    : Bool;
   faultOvr : Bool;   // overrange
   faultUnr : Bool;   // underrange
END_STRUCT
```

**Uso**: ogni canale AI fisico del banco. Popolato da `fcAiRd` con riparametrazione via `t_anChCfg`.

### `t_aoCh` — canale AO standard

```scl
STRUCT
   valEng   : Real;   // valore ingegneristico (es. -100..100 con 0 = idle)
   valPts   : Int;    // valore raw punti (es. 0..27648 per AQ)
   done     : Bool;
   fault    : Bool;
   faultOvr : Bool;
   faultUnr : Bool;
END_STRUCT
```

**Uso**: ogni canale AO fisico del banco. Scritto da `fcAoWr` con riparametrazione via `t_anChCfgAo`.

### `t_anChCfg` — configurazione canale AI (taratura)

```scl
STRUCT
   Tmist  : Real;   // raw min (Gefran: Taratura Min Segnale Trasduttore)
   Tmast  : Real;   // raw max
   TmiFlt : Real;   // raw min con filtro applicato (?)
   TmaFlt : Real;   // raw max con filtro applicato (?)
   ItMist : Real;   // min in unità ingegneristiche (Ingegneristico tmist)
   ItMast : Real;   // max in unità ingegneristiche
   kNew   : Real;   // coefficiente filtro (es. media mobile)
   valSym : Real;   // valore simulato (0.0 = disabilitato)
END_STRUCT
```

**Uso**: per ciascun canale AI, una struttura di calibrazione. Vedi *Convenzioni emerse → Famiglia naming Gefran*.

### `t_anChCfgAo` — configurazione canale AO (naming moderno, no Gefran)

```scl
STRUCT
   EngMin  : Real;   // range minimo ingegneristico
   EngMax  : Real;   // range massimo ingegneristico
   EngIdle : Real;   // valore idle ingegneristico
   PtMin   : Int;    // range minimo punti raw
   PtMax   : Int;    // range massimo punti raw
   PtIdle  : Int;    // valore idle punti raw
   valSym  : Real;   // valore simulato
END_STRUCT
```

**Nota**: scritta dopo aver abbandonato la famiglia Gefran (vedi *fsua* "abortito"). Naming auto-esplicativo. Modello da seguire per tutto il codice nuovo.

### `t_aiMbCh` — canale AI letto via Modbus (semplificato)

```scl
STRUCT
   valPt  : Real;
   valEng : Real;
END_STRUCT
```

Solo valore raw + ingegneristico. Niente fault perché la diagnostica è gestita lato comunicazione Modbus.

### `t_aoMbCh` — canale AO scritto via Modbus

Analogo a `t_aiMbCh`, lato uscite.

### `t_aiProcCComm#pat` — pattern AI di processo "common"

**Pattern**: aggregato di canali AI **comuni a vari modi operativi** del banco (motore trascinamento, drive, segnali condivisi). Nel 0645 ha 15 canali tipo `TTaiSERBTRAS`, `WSrdTRASSPEE`, `BPaiTRASCINA`, `VFar*` (drive trasm.).

**Adattamento**: si riscrive il contenuto con i canali "comuni" del banco specifico. La struttura (aggregato di `Real` con commento per ogni canale) è il pattern; i nomi sono da rifare.

### `t_aiProcCirc#pat` — pattern AI di processo del circuito

**Pattern**: come `t_aiProcCComm#pat` ma per **canali del singolo circuito** (serbatoio, monte/valle UUT, portata, contropressione, angolo termostato). Nel 0645 ha 19 canali (`TTaiSERBATOI`, `BPaiMONTEUUT`, `PTaiANGLABSL`, ecc.).

**Quando ti serve**: in banchi multi-circuito si istanzia uno per circuito (olio, glicole, termostato).

---

## `Base/30-regolazione/`

### `t_PIDstr` — regolatore PID completo

```scl
STRUCT
   Enb, Reverse : Bool;
   SP, PV, Out  : Real;                                     // valori engineering
   P, I, D, KD, DeT : Real;                                 // tuning (scala 0-100%)
   PV_Range_Min, PV_Range_Max : Real;                       // scaling
   Out_Range_Min, Out_Range_Max : Real;
   SP_Perc, PV_Perc, Out_Perc : Real;                       // internals normalizzati
   app_I, DT_D, ERR, Y, BIAS : Real;                        // lavoro PID + anti-windup
END_STRUCT
```

**Uso**: PID standard, ISA con derivata filtrata su PV, anti-windup su BIAS.

**Quando NO**: PID con feed-forward strutturato o MIMO → estendere/avvolgere.

### `t_pwm` — uscita PWM su digital output

```scl
STRUCT
   Percent  : Real;   // duty cycle desiderato
   PeriodS  : Real;   // periodo (secondi)
   ph       : t_ph;   // fase ON/OFF interna
   tmcOn    : Int;    // tempo ON corrente (centesimi)
   tmcOff   : Int;    // tempo OFF corrente (centesimi)
   appOut   : Bool;
   Enable   : Bool;
   Out      : Bool;
END_STRUCT
```

**Uso**: tipicamente in cascata sopra PID per pilotare attuatori on/off temporizzati (resistenze termoregolazione, valvole).

### `t_soc` — SOnda in Corto (diagnostica sensore)

```scl
STRUCT
   ph         : t_ph;
   spTmChange : Real;   // tempo max attesa per la variazione
   spTChange  : Real;   // soglia di variazione attesa sul feedback
   pvPwr      : Real;   // potenza applicata
   pvTCurr    : Real;   // temperatura corrente
   oltTCurr   : Real;   // temperatura riferimento (ingresso finestra)
   soc        : Bool;   // diagnosi: anomalia
   socRst     : Bool;   // reset diagnosi
END_STRUCT
```

**Semantica** (eredità Gefran): se applico potenza ≥ soglia mi aspetto variazione del feedback ≥ `spTChange` entro `spTmChange`. Se **non** accade: potenza mancante, sonda sfilata dal pozzetto, termocoppia con cavo cortocircuitato fuori dal punto di misura (caso più subdolo).

**Pattern riusabile**: "condizione di risposta attesa entro finestra temporale" per diagnosticare guasti silenti. Nome conservato per coerenza con famiglia Gefran (vedi sezione dedicata).

---

## `Base/40-allarmi_monitoraggio/`

### `t_alStr` — struttura allarme completa

```scl
STRUCT
   b          { S7_SetPoint := 'False' } : t_bitCfgAl;   // configurazione bit
   ph         { S7_SetPoint := 'False' } : t_ph;          // stato SM allarme
   elapsed    : UDInt;                                    // tempo dalla generazione
   count      : UDInt;                                    // contatore eventi
   insTmcTrig : UDInt;                                    // istante trigger (centesimi)
   insTmcSp   : UInt;                                     // istante setpoint
END_STRUCT
```

**Uso**: ogni allarme dell'impianto è un'istanza di `t_alStr`. Tipicamente in array dentro `dbAIs` (allarmi istanza). Il bitfield `b` ne configura il comportamento, `ph` ne governa il ciclo di vita.

### `t_bitCfgAl` — bit di configurazione singolo allarme

```scl
STRUCT
   reqd    : Bool;   // required: allarme abilitato
   rstMan  : Bool;   // richiede reset manuale (vs auto al ripristino)
   audible : Bool;   // sirena attiva
   active  : Bool;   // stato attivo corrente
   loPri   : Bool;   // bassa priorità
   blind   : Bool;   // cieco (mascherato in HMI)
   input   : Bool;   // ingresso/sorgente
   live    : Bool;   // valutato sul livello
   liveRe  : Bool;   // valutato sul rising edge
   mem     : Bool;   // memorizzato (latched)
   piuPiu  : Bool;   // beep ripetuto?
END_STRUCT
```

**Uso**: configurazione del comportamento di ciascun allarme (manuale vs auto, con/senza sirena, latched o no, ecc.). Il "linguaggio" di base degli allarmi VBA.

### `t_monEv#pat` — singolo evento monitorato (valvola/motore con feedback)

```scl
STRUCT
   ph          : t_ph;
   inh         : Bool;
   cmd         : Bool;
   q           : Bool;
   fcOff       : Bool;
   fcOn        : Bool;
   qNdx        : Int;     // indice canale uscita        ← da configurare
   fcOffNdx    : Int;     // indice canale feedback OFF  ← da configurare
   fcOnNdx     : Int;     // indice canale feedback ON   ← da configurare
   fault       : Bool;
   errNdx      : Int;     // indice allarme generato     ← da configurare
   graceTime_c : UDInt;   // tolleranza centisecondi
END_STRUCT
```

**Uso**: ogni attuatore con feedback (valvola on/off con finecorsa, contattore con aux). Logica: comando → attesa `graceTime_c` → verifica coerenza `fcOn/fcOff` con `cmd` → in caso negativo `fault=true` + genera allarme `errNdx`.

**Perché `#pat`**: la struttura non cambia, ma **gli indici** (`qNdx`, `fc*Ndx`, `errNdx`) vanno configurati per ogni istanza. Il marker ricorda che l'istanziazione richiede personalizzazione (in init o dichiarazione).

### `t_monEvs#pat` — aggregato di eventi monitorati per il banco

**Pattern**: struct con un membro `t_monEv` per ciascun attuatore monitorato del banco. Nel 0645 ha 5 valvole (`YVuRABBOCCO_x70`, `YVuRICIRCOL_x80`, `YVuNONSVUOTA_x40`, `YVuNOPURGE_TP_x30`, `YVuNODRENUUT_1077`).

**Adattamento**: si riscrive con i monitorati del banco specifico. Convenzione nomi: stessa di `t_di_def#pat` (sigla `YVu` per valvola, descrittore, numero hardware).

---

## `Base/50-comandi/`

### `t_cmd2plc#pat` — comandi PC → PLC (32 bit, 2 Word)

Pattern bitfield **fondamentale**. Naming dei bit segue la convenzione VBA evoluta — ogni bit ha un nome del tipo `<prefisso2-3char><indiceHexNelGruppo><continuazioneSemantica>`. Word 0 (bit 00..0F) tipicamente per comandi generali/allarmi, Word 1 (bit 10..1F) per comandi specifici banco.

Estratto dal 0645:
```scl
STRUCT
   al00armRst    : Bool;   // 0001 - reset allarmi
   al01armAck    : Bool;   // 0002 - tacitazione allarmi
   ge02nRst      : Bool;   // 0004 - reset generale
   bu03sy        : Bool;   // 0008 - busy
   …
   ma07nStart    : Bool;   // 0080 - marcia manuale
   au08toStart   : Bool;   // 0100 - marcia automatica
   …
   co0Enfirm     : Bool;   // 4000 - conferma
   ca0Fncel      : Bool;   // 8000 - annulla
   ol10io        : Bool;   // 0001 - sel. olio (banco-specifico)
   gl11ic        : Bool;   // 0002 - sel. glicole
   …
END_STRUCT
```

**Adattamento**: i bit Word 0 (00..0F) dovrebbero essere **standard VBA** (allarmi/marcia/stop/conferma sempre uguali nei vari banchi); Word 1 (10..1F) è banco-specifica. Si rinominano i bit semantici mantenendo il pattern naming.

### `t_cmd2pc#pat` — feedback PLC → PC (16 bit)

Stesso pattern naming. Nel 0645:
```scl
STRUCT
   ca0ncClOlio  : Bool;
   ca1ncClGlic  : Bool;
   ca2ncAper    : Bool;
   ba3ncpron    : Bool;
   pu4rgDone    : Bool;
   …
   stAepAlarm   : Bool;
   flb..flf     : Bool;   // placeholder per espansione
END_STRUCT
```

**Pattern**: lasciare bit `flb..flf` (o equivalenti) come placeholder per espansione futura, evitando di dover rifare il layout `dbComm`.

### `t_cmdSvc#pat` — bitfield comandi servizio (16 bit)

```scl
STRUCT
   olio, glic, term : Bool;            // selezione circuito (banco-specific)
   riem, spur, circ : Bool;            // riempimento, spurgo, circolazione
   cprs, pres       : Bool;            // contropressione, pressatura
   risc, raff, raffForz : Bool;        // riscaldamento, raffreddamento
   svuo             : Bool;            // svuotamento
   gateOvride       : Bool;            // override portellone
   auxCmd_13..15    : Bool;            // spare
END_STRUCT
```

**Pattern naming**: qui Roberto **non** ha usato il naming bit-hex inline (è "tipo di comandi del servizio banco"), ma nomi semantici diretti. Ammesso in casi dove i nomi sono più importanti dell'ordinamento bit. **Nota**: in Word singola si può comunque ricostruire la posizione bit dal layout di dichiarazione.

### `t_manCmd#pat` — comandi manuali strutturati per layer

Struct ricca con **layer architetturali** marcati nei commenti del sorgente:
- LAYER 1 fluid: `phSvuota`, `phRegPress`, `phSpurgo`, `phTRamp`, `spurgoFatto`, …
- LAYER 2 UUT: `uutEnable`, `uutSpW`
- LAYER 3 contropressione: `cprEnable`, `cprSpVProp`
- LAYER 4 termoregolazione
- LAYER 5 pressatura: `pressEnable`
- STATUS/feedback: `phActLyrFluid`, `activeLayer*`, `timeActive`
- ALLARMI: `alrConflict`, `alrTimeout`, `alrOverpress`, `alrLowLevel`

**Pattern**: i layer sono il **paradigma standard VBA** per comando manuale strutturato. Si adatta il contenuto di ciascun layer al banco specifico, mantenendo la struttura.

### `t_gCmd` — contenitore stabile aggregato comandi

```scl
STRUCT
   req2pc    { S7_SetPoint := 'False' } : t_cmd2pc#pat;
   req2pcSt  { S7_SetPoint := 'False' } : t_cmd2pc#pat;
   req2plc   { S7_SetPoint := 'False' } : t_cmd2plc#pat;
   req2plcSt { S7_SetPoint := 'False' } : t_cmd2plc#pat;
   cMan : t_cmdSvc#pat;
   cAut : t_cmdSvc#pat;
   cLok : t_cmdSvc#pat;
   cWrk : t_cmdSvc#pat;
END_STRUCT
```

**Importante**: `t_gCmd` è un **contenitore stabile senza `#pat`** anche se contiene UDT pattern. Pattern progettuale: i contenitori che aggregano UDT pattern hanno **nome stabile**, e l'adattamento riguarda solo i pattern annidati. Quando si adatta un progetto: si rinominano i `#pat` interni, `t_gCmd` resta `t_gCmd`.

### `t_rstSkipMask#pat` — bit di skip per reset selettivi

```scl
STRUCT
   skipPompOlio       : Bool;
   skipPompGlic       : Bool;
   skipBooster        : Bool;
   skipRiscOlio       : Bool;
   skipRiscGlic       : Bool;
   skipRaffOlio       : Bool;
   skipRaffGlic       : Bool;
   skipTrascinamento  : Bool;
   skipAllEvMon       : Bool;   // skip tutte le valvole monitorate
END_STRUCT
```

**Uso**: maschera per il reset generale: bit a 1 = "questo elemento NON viene resettato". Permette procedure di reset granulari.

**Adattamento**: si riscrive con gli elementi resettabili del banco specifico (motori, riscaldatori, pompe, valvole monitorate, ecc.).

---

## `Base/60-setpoint_base/`

Pattern di setpoint generici. Le specializzazioni progetto-specifiche (`t_spCircuito`, `t_spTras`, `t_spWk`) vivono in `Progetto/SetPoints/`.

### `t_spCfg#pat`, `t_spGrp#pat`, `t_spLiv#pat`, `t_spTm#pat`, `t_spTo#pat`

Famiglia di setpoint **per categoria di parametro**:
- `t_spCfg#pat` — configurazione generale (es. `noNumRipet : Int` numero ripetizioni avvio)
- `t_spGrp#pat` — raggruppamento generico
- `t_spLiv#pat` — livelli (alti/bassi/soglie)
- `t_spTm#pat` — tempi/durate/ritardi (es. `duraBeepStart`, `pausaSeq`, `pausaRipet`)
- `t_spTo#pat` — timeout di sicurezza

Ogni progetto **estende/adatta** queste struct con i campi e i default specifici. Il marker `#pat` ricorda di personalizzare.

**Pattern di default value**: usare `:= valore` nella dichiarazione per default ragionevoli (es. `duraBeepStart : UInt := 50`). Riduce gli init runtime.

### `t_stepSp` — setpoint complesso con qualità e sicurezza

```scl
STRUCT
   sp     : Real;   // setpoint valore
   lqcl   : Real;   // Lower Quality Control Limit (tolleranza inferiore)
   uqcl   : Real;   // Upper Quality Control Limit
   lsl    : Real;   // Lower Security Limit (arresto banco)
   usl    : Real;   // Upper Security Limit
   status { S7_SetPoint := 'False' } : t_stepSpStatus;
END_STRUCT
```

**Uso**: ogni setpoint di un passo ciclo prova ha 5 livelli: valore + 2 limiti qualità (warning) + 2 limiti sicurezza (arresto). Più la word di status che indica in tempo reale dove si trova la PV.

**Senza `#pat`**: pattern strutturalmente stabile.

### `t_stepSpStatus` — word stato 16 bit per `t_stepSp`

```scl
STRUCT
   in0range : Bool;
   lq1alm   : Bool;   // lower quality alarm
   hq2alm   : Bool;   // higher quality alarm
   ls3alm   : Bool;   // lower security alarm
   hs4alm   : Bool;   // higher security alarm
   fl5..flF : Bool;   // placeholder per espansione
END_STRUCT
```

**Pattern naming**: `<prefisso><indiceHex><semantica>` come per i comandi. I primi 5 bit sono semanticamente definiti, gli `fl5..flF` riservati per estensione futura.

---

## `Base/70-termoregolazione/`

### `t_trBase` — termoregolazione base (riscaldamento + raffreddamento)

Struttura ampia con:
- 2 phase handler: `phH` (heating), `phC` (cooling)
- 2 PID: `pidRis` (riscaldatore), `pidRaf` (raffreddatore)
- 2 PWM: `pwmRisc`, `pwmRaf` (cascata sopra PID)
- 1 SOC: `socRis` (diagnostica sonda riscaldamento)
- Setpoint: `spAut`, `spMan`, `spAct`, `spDeadBand`
- Comandi: `iRun`, `arrImmed`, `flInhTreg`, `flEnabRisc`, `flEnabRaff`, `iFaultReset`
- Status: `uFault`
- Aux Q (uscite ausiliarie configurabili): `auxQPotenza`, `auxQModulante`, `auxQRaffr`, `auxQSoffiante`, `auxQVentCella`, `auxQFreddoGlic`, `auxQFreddoGas`, `auxQSvuoGlic`, `auxQCompr`
- Aux I (ingressi ausiliari): `auxIAllLpComp`, `auxIAllFlusso`
- Comando reverse: `uReqPCella`

**Uso**: cuore della termoregolazione. Le aux Q/I permettono di pilotare attuatori secondari (ventola, soffiante, valvole frigo) e leggere allarmi specifici (low pressure compressore, mancanza flusso).

### `t_tRegFl` — termoregolazione fluido (specializzazione)

```scl
STRUCT
   phI               { S7_SetPoint := 'False' } : t_ph;     // phase iniziale
   tr                : t_trBase;                            // composizione, non ereditarietà
   spMinQRisc        : Real := 10.0;
   spFsOvert         : Real := 0.0115;
   spTmcRitInsRisc   : Int  := 500;     // ritardo inserzione riscaldamento (centesimi)
   spTmcInsensQ0     : Int  := 400;
   spPeriodoAttivo   : Int  := 5;
   FLiRiempOk        : Bool;
   MPiCircrun        : Bool;
   MPiCircfault      : Bool;
   MPqRaffRdy        : Bool;
   MPqRaffRun        : Bool;
   spTMaxAss         { S7_SetPoint := 'True' } : Real := 120.0;   // °C - taglio hardware
END_STRUCT
```

**Uso**: termoregolazione di un fluido specifico, costruita sopra `t_trBase`. Contiene parametri di safety (`spTMaxAss` taglio hardware), interlock (`FLiRiempOk` solo se circuito riempito), e segnali ausiliari della parte fluidica.

---

## `Base/A0-devices/<Vendor>/`

Wrapper VBA standard per device esterni che VBA usa abitualmente. Pattern: una sottocartella per famiglia (`DeltaDrives/`, in futuro `SiemensVFD/`, `BeckhoffIO/`, ecc.).

### `Base/A0-devices/DeltaDrives/` — driver Delta (C2000+, VFD-EL, ecc.)

#### Pattern di decomposizione standard (5+1)

`t_DeltaDrv` aggrega 5 sotto-struct + 1 flag:
```scl
STRUCT
   ph     { S7_SetPoint := 'False' } : t_ph;             // stato SM driver
   eng    : t_DeltaDrvEng;                                // valori engineering
   pts    : t_DeltaDrvPts;                                // valori raw points
   naif   : t_DeltaDrvNaif;                               // status non decodificato
   sp     : t_DeltaDrvSp;                                 // setpoint/comandi
   onLine { S7_SetPoint := 'True' } : Bool;
END_STRUCT
```

**Pattern raccomandato per qualunque wrapper di device esterno**: separare in **eng** (unità ingegneristiche), **pts** (raw), **naif** (interi non decodificati), **sp** (setpoint/comandi), **ph** (stato), **onLine**. È il modello da seguire per Siemens VFD, Beckhoff, qualunque drive/strumento accessibile via Modbus/Profinet.

#### `t_DeltaDrvEng` — valori in unità ingegneristiche

```scl
STRUCT
   inFreq_Hz   : Real;
   outFreq_Hz  : Real;
   outCurr_A   : Real;
   dcBus_V     : Real;
   acOut_V     : Real;
   outPF_grad  : Real;
   actT_xCento : Real;   // °C per cento (per scaling)
   outPwr_KW   : Real;
END_STRUCT
```

#### `t_DeltaDrvPts` — valori raw points (con UM nei nomi)

```scl
STRUCT
   inFreq_cHz  : Int;    // centi Hz
   outFreq_cHz : Int;
   outCurr_cA  : Int;    // centi A
   dcBus_dV    : Int;    // deci V
   acOut_dV    : Int;
   …
   outPF_dGrad : Int;    // deci gradi
   actT_xMille : Int;    // valore × 1000
   actSpd_rpm  : Int;
   …
   outPwr_hW   : Int;    // hecto W
END_STRUCT
```

**Pattern UM esteso**: prefisso letterale del moltiplicatore SI nel suffisso del nome. Vedi *Convenzioni emerse → Naming UM*.

#### `t_DeltaDrvNaif` — stato non decodificato

Interi grezzi: `almStatus`, `warnStatus`, `mStepSpdSts`, `countVal`, `actSpd_rpm`, `cntPg`, `cntPg2`. Più un membro `status : t_DeltaDrvStatus` per la decodifica bit-per-bit.

#### `t_DeltaDrvStatus` — decodifica bit del Naif (commenti hex)

```scl
STRUCT
   rsFullStop   : Bool;   // 3
   rsStopping   : Bool;   // 3
   rsStandby    : Bool;   // 3
   rsRunning    : Bool;   // 3
   jog          : Bool;   // 4
   dirFwd       : Bool;   // 18
   …
   hoaOff       : Bool;   // e000
   hoaHandOn    : Bool;   // e000
   hoaAutoOn    : Bool;   // e000
   …
END_STRUCT
```

**Pattern documentazione**: il commento di ogni bit riporta la **maschera hex** del campo nel registro raw del drive. Permette debug rapido senza datasheet alla mano.

#### `t_DeltaDrvSp` — setpoint e comandi

```scl
STRUCT
   spSpdMan, spSpd : Real;       // setpoint velocità
   spSpdFreq       : Int;        // setpoint frequenza
   spTorqLim       : Int;        // limite coppia
   run, stop       : Bool;
   fwd, rev        : Bool;
   extFlt          : Bool;       // external fault
   almRst          : Bool;       // alarm reset
   onLine          : Bool;
   cmdCtrl         : Word;       // comando control register
   cmdFlt          : Word;       // comando fault register
END_STRUCT
```

---

## Convenzioni emerse dall'analisi

### Naming — prefissi UDT/blocchi/costanti

| Prefisso | Ruolo | Esempi |
|---|---|---|
| `t_` | tipo dato (UDT) | `t_ph`, `t_di` |
| `fc` | function block ciclico | `fcDiRd`, `fcAoWr` |
| `f_` | funzione pura / helper | `f_pid`, `f_pwm` |
| `db` | data block | `dbB`, `dbAIs` |
| `cfg` | costante utente (configurazione dimensioni) | `cfgNDi`, `cfgNDo` |
| `CP` | costante utente (indice canale mnemonico) | `CPdiXXX`, `CPaoYYY` |
| `pv` / `sp` | process value / setpoint | `pvPresMonte`, `spTemp` |
| `cmd` / `fl` | comando / flag | `cmdStart`, `flDone` |
| `aux` | ausiliario (Q output, I input) | `auxQPotenza`, `auxIAllFlusso` |

### Naming bit con indice hex inline

**Convenzione VBA evoluta** (osservata in `t_cmd2plc#pat`, `t_cmd2pc#pat`, `t_stepSpStatus`, ecc.):

Pattern: `<prefissoSemantico><indiceHexNelGruppo><continuazioneSemantica>`

- Prefisso 2-3 caratteri tipo (`al` allarme, `ge` generale, `bu` busy, `co` comando, `ma` manuale, `au` auto, `cy` cycle, `ca` cancel, `ol` olio, `gl` glicole, …)
- Indice esadecimale del bit nel gruppo (es. word: 00..0F per Word0, 10..1F per Word1; 0..F per singola Word)
- Eventuale completamento semantico (es. `armRst` reset allarmi, `nfirm` conferma)

Esempi: `al00armRst` = bit 0 di Word 0, allarme reset. `ge02nRst` = bit 2 di Word 0, generale reset. `co0Enfirm` = bit 14 (E hex) di Word 0, conferma. `ol10io` = bit 0 di Word 1, selezione olio.

**Vantaggi**: leggibile come parola (`al00armRst` ≈ "alZeroZeroArmRst"), traccia immediata della posizione bit, ammette espansione semantica.

**Pattern bit placeholder**: se un bit non è ancora semanticizzato, usare `flN` o `<prefisso>NN` (es. `flb`, `flc`, `flf` o `aux13`, `aux14`, `aux15`). Quando si rinomina, mantenere la posizione hex.

### Naming UM nelle variabili numeriche

Tre pattern, in ordine di preferenza:

**1. Suffisso dopo `_`** (variabili numeriche con UM esplicita):
- `_c` centisecondi, `_s` secondi, `_ms` millisecondi, `_h` ore (tempo)
- `_V` volt, `_A` ampere, `_Hz` hertz, `_KW` kilowatt
- `_grad` gradi, `_rpm` giri/min

**2. Embedding nel nome** pattern `tm[UM][Semantica]` (storico, in uso):
- `tmIn` = timestamp (no UM, è diff contro clock)
- `TMcElap` = tempo elapsed in centisecondi
- `tmsRun` = tempo in secondi
- `tmcOn`/`tmcOff` = centisecondi ON/OFF

**3. Suffisso moltiplicatore SI** (per scaling raw points, vedi `t_DeltaDrvPts`):
- `_cHz` centi Hz (×0.01)
- `_dV` deci V (×0.1)
- `_hW` hecto W (×100)
- `xCento` per Cento (×100, valore intero che rappresenta valore × 100)
- `xMille` per Mille (×1000)

Lettera dopo underscore o prefisso = **moltiplicatore decimale SI** (`m`=milli, `c`=centi, `d`=deci, `h`=hecto, `K`=kilo). `x` = "per" (moltiplicato per), seguito dal divisore implicito (`xCento` = il valore nominale è ottenuto dividendo per 100).

### Pattern di decomposizione device esterno

`ph + eng + pts + naif + sp + onLine` (vedi `t_DeltaDrv`). Riusabile per qualunque driver/strumento accessibile via fieldbus.

### Pattern contenitore stabile + sub-struct pattern

I contenitori che aggregano UDT `#pat` hanno **nome stabile** (no `#pat` esterno). Es. `t_gCmd` aggrega `t_cmd2pc#pat`/`t_cmd2plc#pat`/`t_cmdSvc#pat` ma resta `t_gCmd`. Nell'adattamento al progetto: rinomina dei `#pat` annidati, contenitore invariato.

### Marker `#pat` — semantica e ciclo di vita

- **`#pat` nel nome del file/UDT** ⇒ "questo tipo è uno **scheletro** o un **template di nomi**: prima di compilare in produzione vanno **rivisti i campi** (rinominati, configurati, eliminati i placeholder)".
- Si usa per: bitfield di comandi con bit semantici banco-specifici; aggregati AI con canali del banco; UDT con indici interni da inizializzare per istanza; setpoint pattern con default da rivedere; definizioni canali DI/DQ con nomi banco.
- **Nel modello** (template VBA) → `#pat` mantenuto.
- **Nel progetto adattato** → `#pat` rimosso quando l'adattamento è completato.
- **Eccezione utile**: alcuni UDT contengono campi placeholder (`flN`, `auxCmd_NN`, `_NN`) per spazio futuro **anche** dopo l'adattamento. Questi rimangono come riserva, e il `#pat` può essere rimosso comunque (la struttura resta valida).

### Famiglia naming di eredità Gefran

Roberto ha portato dagli anni Gefran una famiglia di nomi brevi, criptici ma **internamente coerenti**. Si mantengono per continuità, documentando l'acronimo nel `00_Convenzioni.md` di progetto.

| Nome | Significato | Ruolo |
|---|---|---|
| `soc` | **SO**nda in **C**orto | diagnostica sensore (vedi `t_soc`) |
| `Tmist` | **T**aratura **MI**n **S**egnale **T**rasduttore | raw min ADC al punto di calibrazione |
| `Tmast` | **T**aratura **MA**x **S**egnale **T**rasduttore | raw max ADC |
| `ItMist` | **I**ngegneristico di `Tmist` | min in unità ingegneristiche |
| `ItMast` | **I**ngegneristico di `Tmast` | max in unità ingegneristiche |
| `minua` | **MIN** **U**scita **A**nalogica | min ingegneristico per AO |
| `maxua` | **MAX** **U**scita **A**nalogica | max ingegneristico per AO |
| `fsua` | **F**ondo **S**cala **U**scita **A**nalogica | *abortito*: si suppone `isua=0`, full-scale non implementato |

**Regola**: non introdurre nomi nuovi in questa famiglia. I nomi **VBA moderni** seguono naming UM moderno (`_c/_s/_ms`, suffissi SI, prefissi semantici `pv/sp/cmd/fl`). Vedi `t_anChCfgAo` (moderna, no Gefran) vs `t_anChCfg` (Gefran).

### Attributo `S7_SetPoint`

Su campi che **non** devono essere modificabili a runtime dall'HMI (stato interno SM, storico, ecc.): `{ S7_SetPoint := 'False' }`. Su campi che possono essere setpoint dall'HMI: `{ S7_SetPoint := 'True' }` (esplicito). Pattern già adottato sistematicamente nel 0645, da preservare.

### Costanti utente per dimensionamento

`cfgNDi`, `cfgNDo`, `cfgLastStep`, `cfgNAiCom`, ecc. permettono di adattare array nei `t_di`/`t_dq`/altri senza toccare gli UDT base. Pattern obbligatorio.

---

## Questioni chiuse

1. ~~Naming `T_xxx` vs `t_xxx`~~ → uniformato `t_`, già fatto.
2. ~~Semantica `t_soc`~~ → SOnda in Corto, documentato.
3. ~~`graceTime_c` unità~~ → centisecondi.
4. ~~`tmsRun` unità~~ → secondi.
5. ~~Separazione UDT / DB templates~~ → due documenti distinti.
6. ~~Rinomina `t_soc`~~ → mantenuto per coerenza famiglia Gefran.
7. ~~`t_cBitInt` due taglie~~ → entrambe (`t_cBitInt` 16, `t_cBitIntX` 32).
8. ~~Marker pattern~~ → `#pat`, ciclo di vita: nel modello c'è, nel progetto adattato si rimuove.
9. ~~UDT in Base con campi project-specific~~ → marcati `#pat` (di_def, dq_def, aiProcCComm, aiProcCirc, monEv, monEvs, rstSkipMask).
10. ~~Convenzione naming bit~~ → `<prefisso><indiceHex><semantica>` come da `t_cmd2plc#pat`, formalizzata.
11. ~~Convenzione UM moltiplicatori~~ → suffisso SI dopo `_` o `x<Cento|Mille>`, formalizzata.

## Questioni aperte residue

1. **Naming bit nel `t_cBitIntX` (32-bit)** — tre opzioni proposte, decisione alla prima occasione d'uso reale.
2. **Migrazione a libreria TIA vera** — quando il catalogo è stabile e c'è almeno un secondo progetto che beneficia.
3. **Apertura `catalogo-db-templates.md`** — dopo `convenzioni-progetto-plc.md`, per ereditare le regole.
4. **Cardinalità famiglia setpoint** — `t_spCfg/Grp/Liv/Tm/To#pat` sono 5 categorie. Servono tutte? Mancano? Da rifinire all'uso reale.
5. **Standardizzazione bit Word 0 di `t_cmd2plc#pat`** — i 16 bit di Word 0 dovrebbero essere realmente standard VBA (allarmi/reset/marcia/conferma sempre uguali). Definire la lista canonica e fissarla.
