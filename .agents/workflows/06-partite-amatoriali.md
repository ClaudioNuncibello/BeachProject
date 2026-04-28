# Workflow: Partite Amatoriali
# Invocazione: /06-partite-amatoriali

Leggi le skill `@partite-amatoriali`, `@campi-turni`, `@auth-ruoli`, `@react-patterns`, `@typescript-expert` e `@api-design-principles` prima di procedere.

Il tuo obiettivo è implementare il sistema completo delle partite amatoriali: creazione, tracking set in tempo reale, doppia validazione del risultato e assegnazione automatica dei punti.

## Cosa devi produrre

### 1. Edge Function create-match
Crea `/supabase/functions/create-match/index.ts`. Riceve: `field_id`, `team_a_ids` (array 2 user_id), `team_b_ids` (array 2 user_id). Verifica che tutti i giocatori abbiano `status: 'available'` e siano in struttura oggi. Crea il record in `matches` con `status: 'pending_start'` e manda una notifica in-app ai due capitani per confermare. Leggi `@partite-amatoriali` per la logica completa e `@auth-ruoli` per i controlli di permesso.

### 2. Edge Function confirm-match-start
Crea `/supabase/functions/confirm-match-start/index.ts`. Ogni capitano chiama questa funzione per confermare. Quando entrambi hanno confermato: cambia `status` a `in_progress`, aggiorna i 4 giocatori a `status: 'playing'`, aggiorna il campo a `status: 'playing'`, crea il primo set in `match_sets`. Leggi `@partite-amatoriali` per il flusso completo.

### 3. Hook useMatch
Crea `src/hooks/useMatch.ts`. Gestisce lo stato della partita corrente dell'utente. Espone: `currentMatch`, `currentSet`, `updateScore(team, points)`, `confirmResult()`. Si iscrive in Realtime ai cambiamenti di `matches` e `match_sets` per aggiornare il tabellone live. Leggi `@api-design-principles` per le subscription Realtime su tabelle collegate.

### 4. Edge Function validate-match-result
Crea `/supabase/functions/validate-match-result/index.ts`. Gestisce la doppia validazione: quando un capitano chiama questa funzione setta il suo flag di conferma. Quando entrambi i flag sono `true` chiama `assign-match-points`. Se c'è disaccordo setta `status: 'disputed'` e notifica lo staff. Implementa il timeout di 10 minuti: se un capitano non risponde, il risultato viene accettato automaticamente. Leggi `@partite-amatoriali` per tutti i casi edge.

### 5. Edge Function assign-match-points
Crea `/supabase/functions/assign-match-points/index.ts`. Assegna 10 punti ai vincitori: aggiorna `points_balance` e `total_points_earned` in `profiles`, crea i record in `point_transactions` con `reason: 'match_win'`. Poi invoca la logica di `handle-match-end` del workflow `/05-lista-attesa` per liberare il campo e notificare la coda. Leggi `@partite-amatoriali` per lo schema di `point_transactions`.

### 6. Componente CreateMatchModal
Crea `src/features/partite/CreateMatchModal/index.tsx`. Modal che permette di creare una partita: seleziona il campo (solo campi liberi), seleziona i compagni di squadra e gli avversari (solo utenti presenti in struttura con `status: 'available'`). Bottone conferma che chiama la Edge Function `create-match`. Leggi `@react-patterns` per i pattern di modal e select con dati live.

### 7. Componente MatchScoreboard
Crea `src/features/partite/MatchScoreboard/index.tsx`. Tabellone live della partita in corso. Mostra: nomi delle coppie, punteggio set corrente con bottoni +1 per ogni squadra (aggiornabili da qualsiasi giocatore delle due coppie), set vinti da ogni squadra, storico set precedenti. Si aggiorna in Realtime. Leggi `@react-patterns` per componenti con aggiornamenti ottimistici.

### 8. Componente ValidationConfirm
Crea `src/features/partite/ValidationConfirm/index.tsx`. Dialog mostrato ai capitani quando la partita entra in `pending_validation`. Mostra il risultato finale e due bottoni: "Confermo" e "Contesto". Se un capitano contesta, mostra un campo testo per spiegare il motivo che viene inviato allo staff. Leggi `@react-patterns` per dialog di conferma.

### 9. Pagina Partite
Crea `src/pages/Partite/index.tsx`. Mostra: se l'utente ha una partita in corso → `MatchScoreboard`. Se non ha partite → bottone "Crea partita" che apre `CreateMatchModal` (visibile solo se l'utente ha fatto check-in). Storico delle ultime partite dell'utente con risultato e punti guadagnati.

## Regole da rispettare

- I punti vengono assegnati **solo** dalla Edge Function — mai dalla UI (vedi `@partite-amatoriali`)
- Il tabellone deve essere Realtime — tutti i giocatori vedono l'aggiornamento simultaneamente
- Usa aggiornamenti ottimistici per il punteggio: aggiorna la UI immediatamente, poi sincronizza col DB
- La validazione deve gestire il caso di timeout (10 min) lato server, non lato client
- Proteggi tutte le Edge Functions con `supabase.auth.getUser()` (vedi `@auth-ruoli`)
