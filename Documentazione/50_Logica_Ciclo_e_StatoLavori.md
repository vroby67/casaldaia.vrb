# 50 — Logica di ciclo, analisi bug e stato lavori

## Obiettivi del reboot (cosa chiudere)
La caldaia a regime funziona bene (giorni/settimane in autonomia). Restano aperti:
1. **Sicurezza no-pellet / flameout**: se finisce il pellet la fiamma si spegne ma la SM resta in combustione stabile a piena potenza indefinitamente. Va rilevato lo spegnimento e portata la caldaia in stby/blocco.
2. **Gestione allarmi**: presente ma non blocca e non si propaga.
3. **Ripresa fiamma a tempo**: predisposta, mai collaudata, tempi sbagliati.
4. **Sequenza accensione**: imbastita, tempi inadeguati — da cronometrare/tarare (vedi 80_Runbook).
5. **Warning Telegram**: pulizia cassetto cenere, pulizia caldaia/camino, aggiungere pellet.
6. **Domotica/Alexa**: imbastita, mai resa operativa.

## Decisioni prese con l'utente
- **Rilevamento flameout** = combinazione **pFiamma + tFumi** (pFiamma basso E/O tFumi sotto minimo, con ritardo di insensibilità).
- **Reazione a mancanza pellet** = **spegnimento controllato immediato + allarme** (ventilazione di post-raffreddamento, poi blocco; riaccensione su richiesta). Non tentativi infiniti.
- **Acquisizione tempi accensione** = strumentazione (MQTT/log) + rete di sicurezza `hist[]`/watch LogicLab. Vedi 80_.

## Analisi bug (dettaglio tecnico)

### B1 — Full power a fiamma spenta (il più grave)
In `mainCycle.xplc`, la fase `mcStableBurn` **non rilegge mai la fiamma**: `b.pv.pFiamma` è controllato solo in `mcFlameWait` (accensione). Esaurito il pellet: fiamma muore → `tCaldaia` cala → `pwrMgr` seleziona step 0 (max potenza) → continua a comandare la coclea → resta lì all'infinito. **Manca la supervisione di flameout in combustione.** Fix previsto: in `mcStableBurn` (e affini) controllo flameout pFiamma+tFumi con ritardo → transizione a spegnimento controllato + allarme.

### B2 — Allarme mancanza pellet non scatta (doppio motivo)
1. `gesAllIst.xplc` (che dovrebbe alimentare `b.al.a[n].b.input` dalle condizioni reali) è uno **STUB**: cabla solo `SInTUTTONOK`. `SNaMANCPELL`, `TOaMANCFIAM`, `TOaACCEFALL` non ricevono mai input. Il ramo `ELSE` per giunta azzera tutti gli input e forza `reset := TRUE`.
2. In `alCfg` (kAlm) l'indice 6 (mancanza pellet) = `0x06` → flag **active = FALSE**, quindi `gesAll` lo salta comunque.
Fix: scrivere `gesAllIst` reale (mappare condizioni→input allarmi) e correggere `alCfg`.

### B3 — flameStatus è uno scheletro
`flameStatus.xplc` ha tutte le fasi `fs*` che aspettano 500ms e passano oltre, senza logica. Da riscrivere o assorbire nella supervisione flameout di B1.

### B4 — Ripresa fiamma / accensione con setpoint sbagliati
In `mcInitBurn` il timeout usa `spTVentInBu_pc` (un setpoint **percentuale**) come secondi `×1000`. Nel ramo ripresa (`mcBurnResume`) usa `spQVentInBu_pc` (percentuale) come tempo. Sintomi coerenti con "predisposta mai collaudata". Da rivedere assegnando i setpoint **tempo** corretti.

### B5 — refill si pianta a tramoggia vuota
In `refill.xplc`, se la coclea esterna non riempie (tramoggia esterna vuota), dopo 2 retry va in `rfACkFull` e **si ferma lì**, senza propagare allarme né mettere in stby. Da collegare a B2 (allarme) e B1/decisione no-pellet (stby). Utile distinguere braciere vuoto (`SNePltLevOK`) da tramoggia esterna vuota (refill fallito).

## Priorità operativa
Concordata: **1) strumentare e cronometrare l'accensione** (occasione rara, caldaia fredda) → **2) tarare i tempi** sui dati reali → **3) flameout+stby (B1)** → **4) catena allarmi reale (B2) + propagazione Telegram** → poi ripresa fiamma (B4), warning manutenzione, domotica.
