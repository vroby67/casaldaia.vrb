# casaldaia.vrb

Software di controllo (PLC Elsist) di una **caldaia a pellet domestica**, rinata dopo il guasto irreparabile della scheda di controllo originale: sensoristica e PLC riprogettati da zero, logica scritta in ST su LogicLab con un paradigma ispirato a TIA Portal.

## Stato
Il funzionamento ordinario è buono: una volta accesa, con pellet disponibile e richiesta termica, la caldaia lavora in autonomia per giorni/settimane. Restano aperti diversi aspetti (vedi `Documentazione/50_Logica_Ciclo_e_StatoLavori.md`):
- supervisione fiamma / blocco sicuro in mancanza pellet (oggi va a piena potenza a fiamma spenta);
- gestione allarmi operativa e propagazione su Telegram (oggi non blocca, non propaga);
- ripresa fiamma a tempo (predisposta, mai collaudata);
- taratura tempi sequenza di accensione;
- warning manutenzione (cenere, pulizia caldaia/camino, pellet);
- integrazione domotica/Alexa.

## Struttura repo
- `casaldaiaReboot.cld/` — progetto PLC LogicLab 6 (multifile, sorgente)
- `node-red/` — flow Node-RED (telemetria/logging/Telegram)
- `Documentazione/` — documentazione tecnica (vedi `INDICE.md`)
- `Contesto/` — input immutabili (manuali, datasheet)
- `note progetto/` — lavoro corrente, todo, rapporti
- `_StandardVBA/` — snapshot convenzioni elucubrIAmo (read-only)

## Come orientarsi
Una sessione Claude legge prima `CLAUDE.md`. Per il quadro tecnico: `Documentazione/INDICE.md`. Per cosa è in corso: `note progetto/_todo.md`.
