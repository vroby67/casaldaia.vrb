# CLAUDE.md — casaldaia.vrb

## Contesto progetto
Controllo PLC della **caldaia a pellet di casa** (retrofit dopo guasto della scheda di controllo originale). PLC **Elsist SlimLine Mps046B XUnified** (ELS20, ARM9). Sorgente in `casaldaiaReboot.cld/` — progetto **LogicLab 6 multifile** (file `.xplc` = XML con codice ST in CDATA). **Non è TIA Portal**: è un paradigma TIA-like portato su Elsist.

Panoramica umana in `README.md`, legami esterni in `LINKS.md`, documentazione in `Documentazione/INDICE.md`.

## Convenzioni
Seguono le convenzioni VBA standard (paradigma **elucubrIAmo**). Snapshot locale in `_StandardVBA/` (sorgente di verità: `G:/Drive condivisi/I - iaRoot/elucubrIAmo`, non sempre montato sulle macchine di deploy → usare lo snapshot). Specializzazioni di progetto: `Documentazione/00_Convenzioni.md`.

## Regole operative (hard)
- **Toolchain**: LL6 sulla macchina di sviluppo, LL5 sul PC dedicato di deploy. **Non disinstallare LL6** (è l'unico che apre il progetto multifile). Trappola spazio codice (LL6 compila anche i blocchi non istanziati) e soluzione: `Documentazione/09_Toolchain_e_Build.md`.
- **Workflow modifiche** (deciso con l'utente, modalità MISTA): modifiche **piccole o critiche** (dichiarazioni, nomi, poche righe, cambi delicati) → **suggerire** il punto esatto e il cambiamento, le applica lui nell'IDE. Rifacimenti **estesi** (logica nuova) → editare i `.xplc`, lui ricarica/scarica da LL6. L'import/reimport in LogicLab crea problemi.
- **Sicurezza**: è una caldaia reale. Niente download azzardati. L'accensione da freddo è rara e costosa da ripetere (tempi lunghi di esaurimento combustione e abbattimento temperature).

## Stato lavori e prossimi passi
`note progetto/_todo.md` (file vivo). Obiettivi del reboot e analisi bug: `Documentazione/50_Logica_Ciclo_e_StatoLavori.md`.

## File chiave (sorgente PLC, sotto `casaldaiaReboot.cld/Casaldaia.cld/src/`)
- `tasks.xres` — schedulazione task (Fast 1ms, Slow 5ms, Boot, Back 100ms)
- `Programs/Ciclo/mainCycle.xplc` — state machine principale (fasi `mc*`)
- `Functions/pwrMgr.xplc` — PWM coclea bruciatore / selezione potenza (fasi `cp*`)
- `Programs/Ciclo/refill.xplc` — ricarica coclea esterna (fasi `rf*`)
- `Programs/Ciclo/flameStatus.xplc` — supervisione fiamma (**STUB, da scrivere**)
- `Programs/Aux/gesAll.xplc` — SM stato allarme; `Programs/Ciclo/gesAllIst.xplc` — sorgente input allarmi (**STUB**)
- `Variables/ordinarie/vRt2plc.xplc` — setpoint (RETAIN) e comandi
- `Structures/Globali/t_cenTerm.xplc` — DB globale `b`
- `Programs/Comunicazione/gesMqtt.xplc` — telemetria MQTT (oggi non schedulato); `mbMasterRtu.xplc` — Modbus RTU temperature
