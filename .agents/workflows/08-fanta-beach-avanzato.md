# Workflow: FantaBeach Avanzato — Rosa, Eventi e Classifica
# Invocazione: /08-fanta-beach-avanzato

> 🔜 **COMING SOON — Da implementare in v2+.**
> Prerequisito: workflow `/07-fanta-beach-base` completato. Non eseguire finché non viene rimosso l'avviso da `@project-context`.

Leggi le skill `@fanta-beach`, `@auth-ruoli`, `@react-patterns`, `@typescript-expert`, `@api-design-principles` e `@security-auditor` prima di procedere.

Il tuo obiettivo è implementare la parte avanzata del FantaBeach: la gestione della rosa (3 coppie), gli eventi FantaBeach creati dagli admin, lo schieramento pre-evento e la classifica globale.

> ⚠️ **Gli eventi FantaBeach sono creati manualmente dagli admin.** Non sono automatici, non sono collegati ai Tornei fisici, e i punti vengono assegnati manualmente dall'admin dopo l'evento in base a competizioni reali esterne all'app. Non c'è un bracket o un calcolo automatico.

## Cosa devi produrre

### 1. Hook useRoster
Crea `src/hooks/useRoster.ts`. Gestisce le 3 coppie dell'utente. Espone: `roster` (array di 3 slot), `updateCouple(slot, cardAId, cardBId)`, `renameCouple(slot, name)`. Valida che una carta non possa essere usata in due coppie contemporaneamente. Leggi `@fanta-beach` per lo schema di `user_roster`.

### 2. Componente RosterEditor
Crea `src/features/fanta-beach/RosterEditor/index.tsx`. Interfaccia per gestire le 3 coppie. Ogni slot coppia mostra: nome coppia (modificabile inline), due slot carta con immagine (o placeholder se vuoto). Cliccando uno slot carta si apre un picker che mostra solo le carte possedute e non già usate in altre coppie. Leggi `@react-patterns` per picker pattern su mobile e `@fanta-beach` per la logica unicità carta per slot.

### 3. Edge Function close-fanta-event-lineups
Crea `/supabase/functions/close-fanta-event-lineups/index.ts`. Invocata dagli admin quando la `lineup_deadline` è scaduta oppure manualmente. Setta `fanta_events.status: 'in_progress'`. Blocca definitivamente gli schieramenti per quell'evento. Solo admin possono invocarla. Leggi `@auth-ruoli` per il controllo ruolo e `@fanta-beach` per lo schema.

### 4. Edge Function award-fanta-event-points
Crea `/supabase/functions/award-fanta-event-points/index.ts`. Invocata dall'admin dall'interfaccia `FantaEventAdminPanel`. Riceve: `fanta_event_id`, `results` (array di `{user_id, points_awarded, notes?}`). Per ogni utente:
1. Crea il record in `fanta_event_results`
2. Aggiorna `points_balance` e `total_points_earned` in `profiles`
3. Crea il record in `point_transactions` con `reason: 'fanta_event'`
4. Setta `fanta_events.status: 'completed'` dopo aver processato tutti
I punti sono assegnati manualmente dall'admin — **non c'è una tabella di punteggi fissi**. Usa `@security-auditor` per verificare che solo gli admin possano invocarla. Leggi `@fanta-beach` per lo schema di `fanta_event_results`.

### 5. Hook useFantaEvents
Crea `src/hooks/useFantaEvents.ts`. Carica la lista degli eventi FantaBeach. Espone: `events` (upcoming / lineup_open / in_progress / completed), `loading`. Per ogni evento carica anche lo schieramento corrente dell'utente (`fanta_event_lineups`). Leggi `@fanta-beach` per gli stati del ciclo di vita.

### 6. Componente LineupSelector
Crea `src/features/fanta-beach/LineupSelector/index.tsx`. Mostrato quando `fanta_events.status: 'lineup_open'` e prima della `lineup_deadline`. L'utente vede le sue 3 coppie dalla rosa e sceglie quale schierare per l'evento. Se lo schieramento è già stato fatto, mostra la coppia schierata con un badge "Schierata ✓" e permette di cambiarla finché la deadline non è scaduta. Se `status` è `in_progress` o `completed`, mostra la coppia schierata in sola lettura. Leggi `@fanta-beach` per la logica di schieramento e il campo `lineup_deadline`.

### 7. Componente FantaEventList
Crea `src/features/fanta-beach/FantaEventList/index.tsx`. Lista eventi FantaBeach raggruppati per stato: "In corso", "Prossimi", "Completati". Ogni evento è una card con nome, data, stato e — se `lineup_open` — un CTA "Schiera la tua coppia". Leggi `@react-patterns` per liste raggruppate.

### 8. Componente FantaEventAdminPanel
Crea `src/features/fanta-beach/FantaEventAdminPanel/index.tsx`. Visibile solo ad `admin`. Mostra tutti gli utenti che hanno schierato per un evento con il loro schieramento. Ogni utente ha un campo input numerico "Punti" e uno "Note (opzionale)". L'admin compila tutti i campi e preme "Assegna punti" → salva in bulk chiamando `award-fanta-event-points`. Leggi `@fanta-beach` per il flusso bulk e `@auth-ruoli` per proteggere il pannello.

### 9. Componente Leaderboard
Crea `src/features/fanta-beach/Leaderboard/index.tsx`. Classifica globale ordinata per `total_points_earned` DESC. Ogni riga: posizione (con medaglia 🥇🥈🥉 per top 3), avatar, username, punti totali guadagnati. L'utente loggato è sempre evidenziato anche se non è in top. Aggiungi un commento in italiano nel codice: la classifica usa `total_points_earned` (mai scalato dagli acquisti), non `points_balance`. Leggi `@api-design-principles` per la query con ranking e `@react-patterns` per liste con evidenziazione elemento corrente.

### 10. Aggiornamento Pagina FantaBeach
Aggiorna `src/pages/FantaBeach/index.tsx` del workflow `/07-fanta-beach-base`. Attiva il tab "Rosa" con il componente `RosterEditor`. Aggiungi un tab "Classifica" con `Leaderboard`. Aggiungi una sezione "Eventi" con `FantaEventList` e il `LineupSelector` per l'evento in corso (se presente). Per gli admin aggiungi accesso al `FantaEventAdminPanel`.

## Regole da rispettare

- I punti degli eventi FantaBeach vengono assegnati **solo** dall'admin tramite `award-fanta-event-points` — non c'è assegnazione automatica
- Solo admin possono creare eventi FantaBeach e assegnare punti — usa `@security-auditor` per verificare
- **Gli eventi FantaBeach NON sono tornei fisici** — non creare bracket, non collegare a competizioni fisiche
- La classifica è basata su `total_points_earned`, non su `points_balance` — commenta questo in italiano nel codice
- `point_transactions` mai esposto in insert/update dalla UI — solo Edge Functions con service_role
- Usa `@typescript-expert` per tipizzare correttamente la struttura di `fanta_event_lineups` e `fanta_event_results`
