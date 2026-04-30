---
name: project-context
description: Fornisce il contesto globale dell'app Beach Volley Catania. Usa questa skill all'inizio di ogni sessione di lavoro o quando hai bisogno di capire la visione generale del progetto, le macro-sezioni, o le decisioni architetturali già prese.
---

# Beach Volley Catania — Contesto Progetto

## Cos'è l'app

Una **web app mobile-first** (PWA) per la community di beach volley di una struttura a Catania. Ha quattro sezioni distinte e indipendenti:

| Sezione | Stato v1 |
|---|---|
| **La Struttura** — campi, check-in, coda turni | ✅ In scope |
| **Partite** — match amatoriali, punteggio, punti ranked | ✅ In scope |
| **Tornei** — eventi fisici con bracket e punti | 🔜 Coming Soon |
| **FantaBeach** — gioco virtuale stile fantacalcio | 🔜 Coming Soon |

> [!IMPORTANT]
> **Campi e partite sono concetti separati.** Occupare un campo (via coda) non implica automaticamente creare una partita nell'app. Le partite si creano solo quando tutti e 4 i giocatori lo decidono esplicitamente.

> [!IMPORTANT]
> **FantaBeach è virtuale.** Non è collegato ai tornei fisici. Gli admin creano "eventi FantaBeach" indipendenti, raccolgono le formazioni degli utenti e assegnano i punti manualmente in base a competizioni esterne all'app.

## Utenti

Solo **giocatori abituali della struttura**. Nessun pubblico esterno nella fase iniziale.

Ruoli (definiti in `auth-ruoli`):
- `player` — utente standard
- `staff` — moderatori della struttura; possono gestire tornei e risolvere dispute
- `admin` — controllo completo; può creare tornei, eventi FantaBeach, assegnare punti e cambiare ruoli

## Le 4 macro-sezioni

### 1. La Struttura (Home operativa)
- Dashboard presenze: chi è in struttura oggi, su quale campo
- Check-in/out: l'utente segnala arrivo e disponibilità
- Mappa campi live: stato di ogni campo (libero / occupato + chi sta giocando)
- Lista d'attesa: coda per ogni campo, con posizione visibile
- Occupare un campo = prendere il turno dalla coda; non richiede la creazione di una partita

### 2. Partite Amatoriali
- Creazione partita su iniziativa di tutti e 4 i giocatori presenti
- Tracking set in tempo reale
- Doppia validazione del risultato (entrambi i capitani devono confermare)
- **Una sola partita "ranked" al giorno per utente** → assegna punti ai vincitori
- Le partite successive nella stessa giornata sono `unranked` (nessun punto, solo per divertimento)
- Le partite NON aggiornano lo stato del campo (sono concetti indipendenti)

### 3. Tornei Ufficiali — 🔜 Coming Soon
> Non implementare nella v1. Mostrare placeholder "Coming Soon" nella navigazione.
- Creazione torneo da parte dei moderatori: nome, data, formato
- Bracket/tabellone aggiornato in real-time dallo staff
- Piazzamento finale → punti assegnati ai giocatori partecipanti

### 4. FantaBeach — 🔜 Coming Soon
> Non implementare nella v1. Mostrare placeholder "Coming Soon" nella navigazione.
- Gioco virtuale stile fantacalcio, completamente separato dai tornei fisici
- Carte collezionabili, shop, rosa, eventi FantaBeach con assegnazione punti manuale dagli admin
- La skill `fanta-beach` documenta la visione futura per riferimento

## Sistema Punti — v1

Nella v1 i punti derivano solo dalle partite amatoriali:

| Fonte                   | Punti | Note |
|-------------------------|-------|------|
| Partita ranked vinta    | +10   | Solo 1 ranked al giorno per utente |
| Partita ranked persa    | +0    | Nessuna penalità |
| Partita unranked        | +0    | Nessun punto, solo per divertimento |

Tornei e FantaBeach aggiungeranno nuove fonti di punti nelle versioni future.

- `total_points_earned` → classifica globale (non scende mai)
- `points_balance` → saldo futuro per lo shop carte (non usato nella v1)

## Decisioni architetturali già prese

- Web app (non nativa), mobile-first, installabile come PWA
- Supabase per tutto il backend (Auth, DB, Realtime, Storage)
- Vercel per il deploy
- React + Vite + TypeScript + Tailwind
- Nessuna doppia valuta: i punti sono unici (acquisto carte + classifica)
- La classifica si basa su `total_points_earned`, non sul saldo
- I campi e le code sono indipendenti dalle partite
- FantaBeach e Tornei sono documentati nelle rispettive skill ma **non vanno implementati nella v1**
- Nella navigazione, Tornei e FantaBeach mostrano un placeholder "Coming Soon"

## Stato del progetto

- Fase: **progettazione** (nessun codice scritto ancora)
- **v1 in scope**: La Struttura (campi/coda) + Partite Amatoriali
- **v1 out of scope**: Tornei, FantaBeach
- Grafiche carte FantaBeach: in fase concept con un gruppo di amici (per versioni future)
- Nessun sistema digitale preesistente (tutto gestito informalmente)
