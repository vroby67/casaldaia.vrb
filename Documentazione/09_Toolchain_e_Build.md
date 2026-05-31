# 09 — Toolchain e build

## LogicLab 5 vs 6
- **LL5** (v9.1.30): sul **PC dedicato** collegato al PLC. È dove "girava tutto" (sta nello spazio codice). Toolchain di **deploy**.
- **LL6** (v10.0.0): su questa macchina di sviluppo. Il progetto è stato portato via GitHub e **convertito in multifile** (`useXPLCFiles=true`) apposta per editarlo/condividerlo con l'AI.

## Trappola: spazio codice esaurito in LL6
**Osservazione (verità di campo)**: LL6 compila **tutti** i POU presenti nel progetto, anche programmi non schedulati e funzioni mai chiamate. LL5 compilava solo ciò raggiungibile dai task. Stesso sorgente → in LL6 si esaurisce lo spazio codice del PLC; in LL5 no.

**Codice morto** (confermato dal report `Build/casaldaiaReboot.unu.xml`, sezione `unuObjects`):
- Programmi non in alcun task: `gesMqtt` (**pesante**: linka `MQTTClient_v3 + SysTCPClient + FIFOFile_v1 + JSONEncoder`), `_storico`, `gesLamp`.
- Funzioni mai chiamate (`F:MAIN:`): `boolToUdint`, `fPid`, `fPwm`, `getPercErog`, `fSoc`, `intToBoolArr`, `boolArrToInt`.

## Soluzioni (in ordine)
**Route A — preferita, niente cancellazioni**: in LL6 `Project → Options → Code generation`, disattivare il flag che corrisponde a **`unusedObjs="true"`** nel `.plcprj` ("compila oggetti inutilizzati"). Ricompilare e controllare il code size. Riporta al comportamento LL5 **mantenendo `gesMqtt`** nel progetto (pronto per l'MQTT futuro) senza che pesi finché non è schedulato.

**Route B — se A non basta**: cancellare dall'albero IDE (tasto destro → Delete, non a mano sui file) gli oggetti dichiarati morti (le 7 funzioni + `_storico`/`gesLamp`); `gesMqtt` **archiviarlo** (spostarlo fuori progetto, resta in git) invece di cancellarlo.

## Installazione LL5 accanto a LL6 — NON farlo
LL5 e LL6 condividono lo stesso product code Elsist: l'installer LL5 vede LL6 come stesso prodotto e propone update invece di affiancarle. **Non disinstallare LL6** per mettere LL5 (perderesti l'ambiente che apre il multifile). Se serve LL5, usarlo dove già gira (PC dedicato via RustDesk, o installarci VS Code+Claude). La strada giusta è far stare LL6 nello spazio (Route A) e lavorare da una sola macchina.

## Note build dal `.plcprj`
- Target: `Mps046_XUnified_1_0` (SlimLine Mps046B XUnified), risorsa `ELS20`/ARM9.
- Libreria: `C:\Program Files (x86)\Elsist\LogicLab\Libraries\Pck055a030`.
- Modbus TCP slave del PLC: `192.168.1.138:502`.
- Errori name-mismatch (es. `C16393`): in LL6 il nome dell'oggetto e la dichiarazione `PROGRAM <nome>` devono coincidere (case-sensitive). Capitato su `gesMqtt`/`gesMQTT`.
