# Struttura locale del Lead — drive `B - Lead`

**Stato**: bozza 1, maggio 2026 — **bozza avanti su `Qualità/`**. Documento nuovo, nato con la separazione Lead/Commessa. Nomenclatura cartelle **ancora da consolidare** (vedi §4).
**Scopo**: definire dove vivono i documenti di un **Lead** (pre-vendita) sul filesystem, in coerenza con l'entità Lead del gestionale RoadMapp, e cosa migra verso la Commessa al momento della conversione.
**Correlati**:
- [../procedura/flusso-commessa.md](../procedura/flusso-commessa.md) — flusso Lead → Commessa (§1-2)
- [../procedura/interfaccia-roadmapp.md](../procedura/interfaccia-roadmapp.md) — azioni gestionale sull'entità Lead
- [struttura-commessa.md](struttura-commessa.md) — struttura della Commessa (drive `A - Commesse`)
- [profilo-macchina.md](profilo-macchina.md) — risoluzione di `roots.lead`

---

## 1. Posizione e principio

Con la revisione di maggio 2026 i **contatti sono separati dalle commesse**. I Lead vivono nel Drive condiviso **`B - Lead`**, distinto da `A - Commesse`.

- **Posizione**: `<lead_root>/...`, dove `<lead_root>` è dichiarato nel [profilo macchina](profilo-macchina.md) (`roots.lead`, tipicamente `<vba_data>/B - Lead`).
- **Numerazione**: il Lead ha un **numero proprio** assegnato dal gestionale (`#NNN`, auto-generato, eventualmente forzabile a mano), **indipendente** dal numero a 4 cifre della Commessa.

**Il Lead è prima di tutto un'entità del gestionale.** Anagrafica, scheda Analisi Fattibilità (7 sezioni) e Offerte vivono **dentro RoadMapp**, non come file. Il drive `B - Lead` ospita solo i **documenti pre-vendita** che non stanno nel gestionale (allegati ricevuti dal cliente, materiale di supporto, bozze).

---

## 2. Ciclo di vita del Lead

```
Creato ──► Analisi ──► Offerta ──► Attesa ordine ──► Convertito ──► [ Commessa in A - Commesse ]
              │                          │
              ├─► Respinto (no-go interno, in analisi)
              └─► Annullato (mancata conversione, causa cliente)
```

- **Creato** — registrazione: titolo, cliente, BU, referente discovery, canale di contatto.
- **Analisi** — scheda Analisi Fattibilità compilata **nel gestionale** (vedi [../modelli/analisi-fattibilita.md](../modelli/analisi-fattibilita.md) come riferimento contenuti). Prerequisito per passare a Offerta.
- **Offerta** — la prima Offerta (entità autonoma) è creata in automatico; se ne possono agganciare più d'una.
- **Attesa ordine** — offerta inviata, in attesa di conferma.
- **Convertito** — terminale positivo: nasce la Commessa, con link `Lead di origine`.
- **Respinto** / **Annullato** — terminali negativi distinti (vedi [flusso-commessa.md §2](../procedura/flusso-commessa.md)).

DoD delle transizioni Lead → [../procedura/dod-checklist.md §1](../procedura/dod-checklist.md#1-flusso-lead).

---

## 3. Contenuto della cartella Lead

Per i Lead che generano documenti pre-vendita su filesystem (non tutti ne hanno: un lead semplice può vivere interamente nel gestionale):

| Contenuto | Natura | Note |
|---|---|---|
| Capitolato / RFQ ricevuto dal cliente | Input | Specifiche, richieste, disegni forniti |
| Materiale di supporto all'analisi | Process | Appunti, preventivi fornitori esplorativi, schemi di massima |
| Bozze di offerta (working) | Process | L'offerta **formale** è gestita come entità in RoadMapp |
| Comunicazioni rilevanti pre-vendita | Archive | Email/documenti significativi della trattativa |

> La scheda Analisi Fattibilità e le Offerte **non** si duplicano su filesystem: stanno nel gestionale (con export PDF on-demand).

---

## 4. Nomenclatura cartella Lead — DA DEFINIRE

> ⚠️ **TODO (prossimo passo)**: la convenzione di nome per le cartelle in `B - Lead` **non è ancora stata decisa**. È uno dei prossimi passi dichiarati (definizione nomenclatura + adozione da parte del team). **Non adottare uno schema senza decisione esplicita.**

Vincoli noti da rispettare quando si deciderà:
- Deve agganciarsi al **numero lead del gestionale** (`#NNN`) come chiave stabile.
- Deve restare **distinta** dallo schema commessa `NNNN - CLIENTE - Descrizione` per non confondere le due entità.
- Deve sopravvivere alla **conversione** (tracciabilità lead → commessa via `Lead di origine`).

Candidati da valutare (nessuno ancora adottato): `NNN - CLIENTE - Titolo`, `LEAD-NNN - ...`, o sola gestione in gestionale senza cartella dedicata per i lead "leggeri".

---

## 5. Conversione: cosa migra verso `A - Commesse`

Alla conversione (vedi [interfaccia-roadmapp.md §2.7](../procedura/interfaccia-roadmapp.md)):

1. Il gestionale crea la **Commessa** (numero a 4 cifre, Tipologia, Profilo) con link `Lead di origine`.
2. Si crea la cartella commessa in `A - Commesse` secondo [struttura-commessa.md](struttura-commessa.md).
3. **Migrano** (copia) verso la commessa i documenti pre-vendita ancora utili:
   - capitolato/RFQ → `Contesto/capitolato/`
   - offerta firmata, ordine cliente → `Commerciale/`
   - comunicazioni chiave → `Commerciale/Email Importanti/`
4. La cartella Lead in `B - Lead` **resta** per tracciabilità storica (non si cancella).

---

## 6. Stato e prossimi passi

**Maggio 2026** — documento nato con la separazione Lead/Commessa. Punti aperti:
- **Nomenclatura `B - Lead`** (§4): da decidere e far adottare al team.
- Conferma di `roots.lead` nei profili macchina (vedi [profilo-macchina.md](profilo-macchina.md)).
- Definizione di quali lead richiedono una cartella su filesystem e quali vivono solo nel gestionale.
