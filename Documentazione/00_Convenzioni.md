# 00 — Convenzioni di progetto

## Sorgente di verità
Convenzioni VBA standard: paradigma **elucubrIAmo**. Snapshot locale in `../_StandardVBA/` (in particolare `plc/convenzioni-progetto-plc.md`). Sorgente originale: `G:/Drive condivisi/I - iaRoot/elucubrIAmo`.

## Specializzazioni / deroghe di questo progetto
- **Non è un progetto TIA né una commessa cliente**: è il PLC della caldaia di casa. Quindi NON si applicano le cartelle da commessa (`Gestione commessa/`, `InterazioniCliente/`, BU/PMC/fattibilità). Si applica solo la parte documentale (CLAUDE/LINKS/README + `Documentazione/` + `note progetto/` + `Contesto/`).
- **Toolchain Elsist LogicLab**, non Siemens TIA. I sorgenti sono `.xplc` (XML con ST in CDATA) nel progetto multifile `casaldaiaReboot.cld/`. Tasks in `tasks.xres`. Setpoint/variabili in `Variables/` (ordinarie + costanti). Strutture (UDT) in `Structures/`.
- **Naming storico del progetto** (precedente alle convenzioni VBA moderne, mantenuto per continuità):
  - I/O con suffisso `_h` (fisico/hardware), `_v` (virtuale), `_f`/`_m` (forzatura valore/stato).
  - Prefissi mnemonici I/O: `CTu*` (contattori/uscite), `SNe*`/`SIe*`/`TSe*`/`FCe*` (ingressi: sensore livello, sicurezza, termostato, finecorsa).
  - Setpoint `sp*` con suffisso UM (`_s` secondi, `_dC` decimi °C, `_pc` percento, `_dk`/`_dK` decimi di K, `_ds` decimi di secondo, `_pm` per mille).
  - Fasi state machine come costanti con prefisso famiglia: `mc*` (mainCycle), `cp*` (caricamento/pwrMgr), `rf*` (refill), `fs*` (flameStatus). Definite in `Variables/costanti/kPh*`.
  - Allarmi: indici `XXnNOME` e flag booleani `XXaNOME` in `t_db45_allIst`; configurazione in costanti `alCfg`/`alRit` (kAlm).
- **Contatori**: variabili `mbCn*` in RAM, tamponate in `cn*` RETAIN; ripristino in `init`.

## Workflow modifiche codice (deciso con l'utente)
Modalità **mista**: piccole/critiche → suggerimento da applicare in IDE; estese → edit diretto dei `.xplc` con reload in LL6. Vedi `CLAUDE.md`.
