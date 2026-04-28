---
name: project-context
description: Fornisce il contesto globale dell'app Beach Volley Catania. Usa questa skill all'inizio di ogni sessione di lavoro o quando hai bisogno di capire la visione generale del progetto, le macro-sezioni, o le decisioni architetturali già prese.
---

# Beach Volley Catania — Contesto Progetto

## Cos'è l'app

Una **web app mobile-first** (PWA) per la community di beach volley di una struttura a Catania. Ha due anime:
1. **Gestione operativa** — campi, presenze, turni, partite amatoriali
2. **FantaBeach** — gioco social con carte collezionabili, rose, tornei

## Utenti

Solo **giocatori abituali della struttura**. Nessun pubblico esterno nella fase iniziale.

## Le 4 macro-sezioni

### 1. La Struttura (Home operativa)
- Dashboard presenze: chi è in struttura oggi, su quale campo
- Check-in/out: l'utente segnala arrivo e disponibilità
- Mappa campi live: stato di ogni campo (libero / occupato + chi gioca)
- Lista d'attesa: coda per ogni campo, con posizione visibile

### 2. Partite Amatoriali
- Creazione partita da parte dei giocatori sul campo
- Tracking set in tempo reale
- Doppia validazione del risultato (entrambi i capitani devono confermare)
- Punti assegnati automaticamente ai vincitori (scala ridotta)

### 3. Tornei Ufficiali (solo Staff/Admin)
- Creazione torneo: nome, data, formato
- Bracket/tabellone aggiornato in real-time dall'arbitro durante l'evento
- Piazzamento finale → punti FantaBeach assegnati automaticamente

### 4. FantaBeach
- Shop carte: acquisto con punti
- Collezione: visualizzazione carte possedute (con grafiche custom in produzione)
- Rosa: creazione di 3 coppie da schierare nei tornei
- Schieramento pre-torneo: scelta della coppia da mandare in campo
- Classifica globale FantaBeach basata su punti totali guadagnati

## Decisioni architetturali già prese

- Web app (non nativa), mobile-first, installabile come PWA
- Supabase per tutto il backend (Auth, DB, Realtime, Storage)
- Vercel per il deploy
- React + Vite + TypeScript + Tailwind
- Nessuna doppia valuta: i punti sono unici (acquisto carte + classifica)
- La classifica si basa su `total_points_earned`, non sul saldo
- Admin + staff della struttura gestiscono i tornei; i player gestiscono le partite amatoriali

## Stato del progetto

- Fase: **progettazione** (nessun codice scritto ancora)
- Grafiche carte FantaBeach: in fase concept con un gruppo di amici
- Nessun sistema digitale preesistente (tutto gestito informalmente)
