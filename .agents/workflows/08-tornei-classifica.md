# Workflow: Rosa, Tornei e Classifica FantaBeach
# Invocazione: /08-tornei-classifica

Leggi le skill `@fanta-beach`, `@auth-ruoli`, `@react-patterns`, `@typescript-expert`, `@api-design-principles` e `@security-auditor` prima di procedere.

Il tuo obiettivo è implementare la parte avanzata del FantaBeach: la gestione della rosa (3 coppie), i tornei ufficiali con bracket real-time, lo schieramento pre-torneo e la classifica globale.

## Cosa devi produrre

### 1. Hook useRoster
Crea `src/hooks/useRoster.ts`. Gestisce le 3 coppie dell'utente. Espone: `roster` (array di 3 slot), `updateCouple(slot, cardAId, cardBId)`, `renameCouple(slot, name)`. Valida che una carta non possa essere usata in due coppie contemporaneamente. Leggi `@fanta-beach` per lo schema di `user_roster`.

### 2. Componente RosterEditor
Crea `src/features/fanta-beach/RosterEditor/index.tsx`. Interfaccia per gestire le 3 coppie. Ogni slot coppia mostra: nome coppia (modificabile), due slot carta con immagine (o placeholder se vuoto). Cliccando uno slot carta si apre un picker che mostra solo le carte possedute e non già usate in altre coppie. Leggi `@react-patterns` per drag & drop o picker pattern su mobile.

### 3. Edge Function lock-tournament-lineups
Crea `/supabase/functions/lock-tournament-lineups/index.ts`. Invocata dallo staff quando il torneo passa a `in_progress`. Setta `locked: true` su tutti i `tournament_lineups` del torneo. Solo staff/admin possono invocarla. Leggi `@auth-ruoli` per il controllo ruolo e `@fanta-beach` per lo schema.

### 4. Edge Function assign-tournament-points
Crea `/supabase/functions/assign-tournament-points/index.ts`. La funzione più complessa del progetto. Leggi attentamente `@fanta-beach` → sezione "Edge Functions necessarie" per la logica completa. Per ogni utente con schieramento nel torneo: legge quale coppia ha schierato, trova il piazzamento di quella coppia in `tournament_results`, assegna i punti corrispondenti (tabella punti in `@fanta-beach`), aggiorna `points_balance` e `total_points_earned`, crea il record in `point_transactions`. Usa `@security-auditor` per verificare che non ci siano vulnerabilità nell'assegnazione punti.

### 5. Hook useTournament
Crea `src/hooks/useTournament.ts`. Carica i dati di un torneo specifico: info generali, bracket (`tournament_matches`), piazzamenti (`tournament_results`). Si iscrive in Realtime ai cambiamenti del bracket durante il torneo live. Espone anche: `myLineup` (la coppia schierata dall'utente), `setLineup(rosterSlot)`, `isLineupLocked`. Leggi `@api-design-principles` per le subscription Realtime su tabelle multiple.

### 6. Componente TournamentBracket
Crea `src/features/fanta-beach/TournamentBracket/index.tsx`. Visualizza il tabellone del torneo. Per ogni partita mostra: le due coppie, il punteggio (se disponibile), il vincitore evidenziato. La vista si aggiorna in Realtime durante l'evento. Gli admin/staff vedono un bottone "Aggiorna risultato" su ogni partita. I player vedono solo la visualizzazione. Leggi `@react-patterns` per componenti con viste condizionali per ruolo e `@auth-ruoli` per leggere il ruolo corrente.

### 7. Componente LineupSelector
Crea `src/features/fanta-beach/LineupSelector/index.tsx`. Mostrato prima dell'inizio torneo (quando `status: 'registration_open'`). L'utente vede le sue 3 coppie e sceglie quale schierare. Mostra i nomi delle carte in ogni coppia. Se `locked: true` mostra la coppia schierata in sola lettura con badge "Schierata ✓". Leggi `@fanta-beach` per la logica di schieramento.

### 8. Pagina Tornei
Crea `src/pages/Tornei/index.tsx`. Lista di tutti i tornei: prossimi, in corso, completati. Ogni torneo è una card cliccabile che porta al dettaglio con `TournamentBracket` e `LineupSelector`. Per staff/admin mostra anche i controlli di gestione: cambia stato torneo, aggiorna risultati partite, conferma piazzamenti finali.

### 9. Componente Leaderboard
Crea `src/features/fanta-beach/Leaderboard/index.tsx`. Classifica globale ordinata per `total_points_earned` DESC. Ogni riga: posizione (con medaglia per top 3), avatar, username, punti totali guadagnati. L'utente loggato è sempre evidenziato anche se non è in top. Leggi `@api-design-principles` per la query con ranking e `@react-patterns` per liste con evidenziazione elemento corrente.

### 10. Integrazione nella Pagina FantaBeach
Aggiorna `src/pages/FantaBeach/index.tsx` del workflow `/07-fanta-beach-base`. Attiva il tab "Rosa" con il componente `RosterEditor`. Aggiungi un quarto tab "Classifica" con `Leaderboard`. Aggiungi una sezione "Tornei" che mostra il torneo in corso (se presente) con il suo `LineupSelector`.

## Regole da rispettare

- I punti del torneo vengono assegnati **solo** dalla Edge Function `assign-tournament-points` — mai dalla UI
- Solo staff/admin possono aggiornare i risultati delle partite torneo — usa `@security-auditor` per verificare
- Il bracket deve essere Realtime durante l'evento — usa la subscription Supabase, non polling
- La classifica è basata su `total_points_earned`, non su `points_balance` — commenta questo in italiano nel codice (vedi `@project-context`)
- Usa `@typescript-expert` per tipizzare correttamente la struttura del bracket (albero di partite)
