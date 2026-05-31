# 30 — Comunicazione

## Modbus RTU master (temperature) — `mbMasterRtu` (task Back)
- Porta **COM2**, 9600 8N1. FB `ModbusMaster_v3`.
- Legge 4 holding register (FC03) ciclicamente dai nodi **2 e 3** (moduli Pt100), `ptrMod` 2→3.
- Valori: `RHRegsInt[]/10.0` → `b.ai[*].valInp` → (via `ripara`) → `b.ai[*].valEng` → `b.pv.tCaldaia/tFumi/tMandata/tRitorno/tEsterna/tInterna/tAriaComb/tCaldaia2`.
- **Importante**: le temperature arrivano da qui, non da `aiRd` (in `aiRd` il blocco temperature è commentato; lì si legge solo `rFiamma`).

## Modbus TCP slave (PLC↔HMI/server)
- PLC slave su `192.168.1.138:502` (vedi `commSettings` nel `.plcprj`). Setpoint RETAIN e comandi scritti da qui.

## MQTT — `gesMqtt` (oggi NON schedulato)
- FB `MQTTClient_v3` + `SysTCPClient` + `FIFOFile_v1`. Pubblica su broker **`10.0.0.107:1883`**, topic **`plc/dati`**, gate `b.cm.flCdataLog`, trigger su `SysClock100`.
- Payload attuale: `tCaldaia, tFumi, tMandata` (REAL) via `JSONEncoder`.
- **Idea instrumentazione** (per tarare accensione/flameout): arricchire il payload con `phMaCy.ptr`, `phMaCy.tmmElap`, `phCari.ptr`, `phRefi.ptr`, `pFiamma`, `ventDuty`, bitmask uscite — tutto castabile a REAL per riusare l'unico `JSONEncoder` collaudato. E **schedulare `gesMqtt` nel task Back**. ATTENZIONE: è il blocco pesante (spazio codice, vedi 09_); valutare contro il budget.
- **Verifica preliminare** (sul docker host, via RustDesk): che `10.0.0.107` sia un'interfaccia di quella macchina e che mosquitto sia in ascolto su quell'interfaccia (non solo localhost); `mosquitto_sub -h localhost -t plc/dati`.

## Node-RED
- Gira in **Docker sotto Windows (WSL)** sulla macchina caldaia. Editor `http://192.168.1.93:1880`. Probabile dual-homed (`10.0.0.107` lato PLC, `192.168.1.93` lato casa).
- `node-red/casaldaiaReboot...` : il `nodeRedFlow.json` versionato nel repo è **vuoto** (il flow reale vive nel container).
- Flow di logging CSV pronto da importare: `node-red/casaldaia-logging-flow.json` (MQTT in `plc/dati` → function → file CSV; aggiustare host broker e path file montato).

## Telegram (da fare)
- Oggi gli allarmi/warning **non** vengono propagati. Due strade:
  1. **Via Node-RED**: il PLC pubblica stato allarmi su MQTT, un flow inoltra a Telegram. (Dipende da Node-RED vivo.)
  2. **Nativo dal PLC**: la libreria Elsist espone `Telegram_v1` (`eLLabHTTPLib`) e `RESTClient_v7`. Possibile inviare direttamente dal PLC, senza Node-RED. Costa spazio codice (pesare contro il budget liberato in 09_).
