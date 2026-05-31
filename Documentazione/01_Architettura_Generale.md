# 01 — Architettura generale del software

Sorgenti in `casaldaiaReboot.cld/Casaldaia.cld/src/`. PLC Elsist SlimLine Mps046B XUnified (ELS20/ARM9).

## Task (`tasks.xres`)
| Task | Periodo | Programmi (in ordine) |
|---|---|---|
| **Fast** | 1 ms | `gesVent` (PWM ventilazione) |
| **Slow** | 5 ms (impostato a 5000µs in init) | `diRd`, `aiRd`, `logiGen`, `mainCycle`, `flameStatus`, `doWr`, `refill`, `gesMand`, `gesAll`, `gesAllIst` |
| **Boot** | single | `init` |
| **Back** | 100 ms | `gesCom2plc`, `tarana`, `mbMasterRtu`, `ManInit`, `bkCalc`, `gesCom2hmi` |

`gesMqtt`, `gesLamp`, `_storico` **non sono schedulati** (vedi 09_Toolchain per le implicazioni spazio codice).

## DB globale `b : t_cenTerm` (in `vGlobal`)
Struttura aggregante (`Structures/Globali/t_cenTerm.xplc`):
- `b.pv` (`t_aiProc`) — process value: `tCaldaia, tFumi, tMandata, tRitorno, tEsterna, tInterna, tAriaComb, tCaldaia2, pFiamma` (tutti REAL).
- `b.cm` (`t_cmdReq`) — comandi/richieste (es. `ru0nRisc`, `ru1nAcs`, `flCdataLog`, `mcStblBurn`, `mcFNextStp`).
- `b.bp` (`t_burner`) — bruciatore + **puntatori fase**: `phMaCy, phFlSt, phCari, phRefi` (tipo `t_ph`), config potenza `regPar[0..3]`/`phLowPwr`/`phMidPwr`/`phHighPwr` (`t_pwrStep`), `ventDuty`.
- `b.tr` (`t_tRegT`) — termoregolazione (`regPar` corrente).
- `b.gm` (`t_gesMand`) — gestione mandata (es. `spTHtngPmp`, `spTDhwPump`).
- `b.au` (`t_auxGen`), `b.bk` (`t_bkCalc`).
- `b.al` (`t_db45_allIst`) — allarmi (vettore `a[0..31]` + flag + indici nominativi).
- `b.ai[0..9]` (`t_aiCh`) — canali analogici grezzi→engineering.

## State machine (puntatori `t_ph`, step a multipli di 5)
`t_ph` contiene `ptr` (fase corr.), `oldPtr`, `tmmIn`, `tmmElap`, e un **anello storico `hist[0..10]`** (`phNo`+`lasted`) aggiornato automaticamente da `ptTm` ad ogni cambio fase → **le durate di ogni fase sono registrate da sole**. `ptSet(ph, n)` setta `ptr := n + dbg`. `ptTm(ph, SysTime)` aggiorna tempi e storico.

- **`mainCycle`** → `b.bp.phMaCy` (`mc*`): `notRdy(0)→cmdWait(5)→initLoad(10)→flameWait(15)→initBurn(20)→noVentIgn(25)→ventRstrt(30)→pwrSel(35)→stableBurn(45)↔ashRemoval(50)`; ramo idle/ripresa: `idleFire(55)→burnResume(60)→ventStart(65)→pelletFeed(70)→ventHold(75)→idleReturn(80)`. Salto diretto a stabile via `mcPrsStabBurn(40)` (debug/trasmissione SW). Chiama `pwrMgr()` in stableBurn. Costanti in `kPhMainCycle` (incl. range anomalie 100-200).
- **`pwrMgr`** (funzione) → `b.bp.phCari` (`cp*`): PWM coclea bruciatore. Sceglie `ptrPwrStep` (0=max,1=media,2=bassa) confrontando `tCaldaia` con `spKTempCald − spKCampRidu`. PWM con periodo `spTPeriCocl_ds` e duty `regPar.pelletPerc`/`ventPerc`. Incrementa contatori funzionamento `mbCnMFunz[]`.
- **`refill`** → `b.bp.phRefi` (`rf*`): quando cade `SNePltLevOK_v` carica la coclea esterna a tempo (`spTCariCExt_s`), con retry. (Vedi bug 50_.)
- **`flameStatus`** → `b.bp.phFlSt` (`fs*`): **STUB** — fasi a vuoto wait 500ms.

## I/O (`Variables/ordinarie/vInputs.xplc`, `vOutputs.xplc`)
Immagini parallele: `_h` (fisico), `_v` (virtuale, ciò che la logica usa), `_f`/`_m` (forzatura valore/stato).
- **Ingressi** (DI %IX2.x): `SIeTutto_OK` (comandi inseriti), `SNePltLevOK` (livello pellet braciere), `TSeOverTemp` (termostato sovratemp.), `SNeBurnConn`, `FCeValvOpen`, `FCeValvClos`.
- **Uscite** (DQ %QX2.x): `CTuExtAuger` (coclea esterna), `CTuBrnAuger` (coclea bruciatore), `CTuIgnHeatr` (candeletta), `CTuHtngPump` (pompa riscaldamento), `CTuDhwPump` (pompa ACS), `FCuMixVOpen`/`FCuMixVClse` (valvola miscelatrice), `CTuVentCmd` (ventilazione; duty in `b.bp.ventDuty`, **% inverso**).
- **Fiamma**: AI `rFiamma` letta in `aiRd` (`AD_VOLT_0_10_COMMON`) → `b.pv.pFiamma` (REAL, % radiazione). Soglia presenza fiamma: `spRPresFiam_pm`/100.
- **Temperature**: lette via Modbus RTU in `mbMasterRtu` (vedi 30_), NON in aiRd (lì il blocco temperature è commentato).

## Setpoint (`Variables/ordinarie/vRt2plc.xplc`)
Quasi tutti **RETAIN** (in memoria tamponata, impostati da HMI/server via Modbus, **non nel sorgente**). `init`/`ManInit` ne fissano solo alcuni di default (tarature analogiche, overtemp, soc, wdog). Tempi chiave sequenza: `spTCariFred_s, spTRiVeIgni_s, spQVentIgni_pc, spTVentInBu_pc, spTPropFiam_s, spTFunzMinI_s, spTPeriCocl_ds, spTToutFiam_s, spTMancAcce_s, spTCariRipr_s, spTVentRipr_s, spQVentRipr_pc, spTCariCExt_s`. Temperature: `spKTempCald_dC, spKIsteCald_dK, spKCampRidu_dk, spKMaxTFumi_dC, spKMinTFumi_dC`. → **Per giudicare i tempi servono i valori reali letti dal PLC** (vedi 80_Runbook).

## Allarmi (sintesi; dettaglio bug in 50_)
Vettore `b.al.a[0..31]` (`t_alStr`: bit config `t_bitCfgAl`, fase `t_ph`, `count`, `insTmmSp`). `gesAll` = SM stato allarme (riposo→ritardo→sonoro/lampeggio→attesa reset). Configurazione caricata in `init` da costanti `alCfg`/`alRit` (kAlm): bit `0x01` reqd, `0x02` rstMan, `0x04` audible, `0x08` active, `0x10` loPri, `0x20` blind. **`gesAllIst`** (che dovrebbe alimentare `a[n].b.input` dalle condizioni reali) è uno **STUB**. Indici utili in `t_db45_allIst`: `SNnMANCPELL=6`, `TOnMANCFIAM=5`, `TOnACCEFALL=4`, `TSnPULICENE=11`, `TSnPULICALD=12`.
