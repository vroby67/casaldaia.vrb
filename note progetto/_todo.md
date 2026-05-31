# _todo — casaldaia.vrb (lavoro corrente)

Aggiornato: 2026-05-31

## Stato attuale
- Mappata l'architettura SW completa (vedi `Documentazione/01_`).
- Identificati i bug chiave (vedi `Documentazione/50_`): B1 full-power a fiamma spenta, B2 allarmi stub, B3 flameStatus stub, B4 setpoint sbagliati ripresa/accensione, B5 refill si pianta.
- Bloccante toolchain in corso: **spazio codice esaurito in LL6** (compila i blocchi non istanziati). Soluzione da provare: **Route A** = disattivare `unusedObjs` in Project→Options→Code generation (vedi `Documentazione/09_`).
- Documentazione/contesto travasati nel repo (questo commit) per poter proseguire da altra macchina.

## Prossimi passi (in ordine)
1. **[toolchain]** Provare Route A in LL6; riportare se il flag esiste e il code size usato/disponibile prima/dopo. Se non basta → Route B (cancellare codice morto, archiviare gesMqtt).
2. **[accensione]** Allestire watch LogicLab + annotare setpoint correnti (vedi `Documentazione/80_`). Cronometrare l'accensione da freddo.
3. **[taratura]** Con i tempi reali, correggere i setpoint sequenza e i bug B4.
4. **[sicurezza B1]** Scrivere supervisione flameout (pFiamma+tFumi) in mcStableBurn → spegnimento controllato + allarme.
5. **[allarmi B2]** Scrivere `gesAllIst` reale + correggere `alCfg`; collegare refill/tramoggia vuota (B5).
6. **[telegram]** Propagare allarmi/warning (via Node-RED o `Telegram_v1` nativo PLC).
7. **[warning manutenzione]** cenere / pulizia caldaia-camino / pellet (su contatori `cn*`).
8. **[domotica/alexa]** integrazione.

## Decisioni
- Flameout = pFiamma + tFumi (con ritardo). No-pellet = spegnimento controllato + allarme (no tentativi infiniti).
- Workflow modifiche: mista (piccole/critiche → suggerite; estese → edit file). Vedi `CLAUDE.md`.
- Toolchain target: LL6 unico (su macchina sviluppo), una volta risolto lo spazio.

## Aperti / da verificare
- Broker MQTT `10.0.0.107` raggiungibile dal PLC e mosquitto in ascolto sull'interfaccia giusta?
- `b.cm.flCdataLog` scrivibile da watch/HMI?
- Valori reali dei setpoint tempo (necessari per la taratura).
