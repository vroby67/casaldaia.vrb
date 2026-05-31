# Legami esterni — casaldaia.vrb

**Progetto**: controllo PLC caldaia a pellet domestica (retrofit).
**Descrizione sintetica**: PLC Elsist SlimLine Mps046B XUnified, ST/LogicLab, telemetria MQTT→Node-RED→Telegram.

## Toolchain / macchine
- **Sviluppo**: questa macchina (host `asusvrb`), LogicLab 6 (v10.0.0), VS Code + Claude Code. Apre il progetto multifile `casaldaiaReboot.cld/`.
- **Deploy**: PC dedicato con **LogicLab 5** (v9.1.30), collegato in linea al PLC. Accesso via **RustDesk**. È la postazione che compila+scarica nel PLC (storicamente "girava tutto").
- Vedi `Documentazione/09_Toolchain_e_Build.md` per la trappola spazio codice LL5/LL6.

## Rete / comunicazione
- **PLC**: Modbus TCP slave a `192.168.1.138:502` (vedi `commSettings` nel .plcprj). Modbus RTU **master** su COM2 (9600 8N1) verso moduli Pt100 (nodi 2 e 3) per le temperature.
- **Broker MQTT**: il PLC pubblica su `10.0.0.107:1883`, topic `plc/dati` (vedi `gesMqtt.xplc`). Da verificare che il broker sia in ascolto su quell'interfaccia.
- **Node-RED**: gira in **Docker sotto Windows (WSL)** sulla macchina caldaia. Editor su `http://192.168.1.93:1880` (o `localhost`). Visibile via RustDesk. Probabile macchina dual-homed (NIC `10.0.0.107` lato PLC, `192.168.1.93` lato casa).
- Flow di logging in `node-red/casaldaia-logging-flow.json`.

## Paradigma / convenzioni
- Sorgente di verità: `G:/Drive condivisi/I - iaRoot/elucubrIAmo` (non sempre montato → snapshot in `_StandardVBA/`).

## Workspace file IDE
- (host `asusvrb`) — al momento nessun `.code-workspace` dedicato; il repo si apre direttamente come cartella.

## Contatti
- Software PLC / proprietario impianto: Roberto Visca (stormdev87@gmail.com)
