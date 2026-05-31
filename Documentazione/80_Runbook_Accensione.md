# 80 — Runbook accensione (cronometraggio/taratura)

Scopo: catturare i tempi reali della sequenza d'accensione da freddo (occasione rara) per tarare i setpoint. La caldaia parte da <30°C.

## Rete di sicurezza (cattura indipendente da MQTT)
Il buffer `hist[]` di `t_ph` registra **da solo** la durata di ogni fase conclusa (`phNo`+`lasted` in ms). Con LogicLab online si vede tutto in diretta. Questo cattura i tempi **anche se l'MQTT non raggiunge il broker**.

### Watch list LogicLab (online)
- `b.bp.phMaCy.ptr`, `.oldPtr`, `.tmmElap`, **`b.bp.phMaCy.hist[]`** (durate fasi)
- `b.bp.phCari.ptr`, `b.bp.phRefi.ptr`, `b.bp.phFlSt.ptr`
- `b.pv.pFiamma`, `b.pv.tFumi`, `b.pv.tCaldaia`, `b.bp.ventDuty`
- uscite: `CTuBrnAuger_h`, `CTuIgnHeatr_h`, `CTuVentCmd_h`, `CTuExtAuger_h`
- ingressi: `SNePltLevOK_v`, `SIeTutto_OK_v`

### Setpoint da ANNOTARE prima dell'accensione (sono RETAIN, non nel sorgente)
`spTCariFred_s, spTRiVeIgni_s, spQVentIgni_pc, spQVentInBu_pc, spTVentInBu_pc, spTPropFiam_s, spTFunzMinI_s, spTPeriCocl_ds, spTToutFiam_s, spTMancAcce_s, spTCariRipr_s, spTVentRipr_s, spQVentRipr_pc, spTCariCExt_s, spRPresFiam_pm, spKTempCald_dC, spKIsteCald_dK, spKCampRidu_dk, spKMaxTFumi_dC, spKMinTFumi_dC`.
> Senza i valori reali non si può giudicare quali tempi sono inadeguati.

## Opzione log continuo via MQTT/Node-RED (bonus)
Dà le curve continue di fiamma/fumi (utili per soglie flameout e ripresa). Richiede:
1. Arricchire `gesMqtt` (vedi 30_) e **schedularlo** nel task Back → ATTENZIONE spazio codice (09_): farlo solo dopo aver liberato spazio (Route A).
2. Verificare broker raggiungibile dal PLC (`10.0.0.107:1883`).
3. Importare `node-red/casaldaia-logging-flow.json`, aggiustare host broker + path file.
4. `b.cm.flCdataLog := TRUE` (gate del publish).

## Sequenza attesa (riferimento, fasi mainCycle)
`notRdy(0)→cmdWait(5)→initLoad(10)→flameWait(15)→initBurn(20)→noVentIgn(25)→ventRstrt(30)→pwrSel(35)→stableBurn(45)`. Cronometrare ciascuna transizione e correlare con pFiamma/tFumi/uscite. Tenere pronto `cmdAbort` per interrompere se la sequenza degenera.
