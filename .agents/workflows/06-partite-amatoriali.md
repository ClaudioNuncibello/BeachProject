# Workflow: Partite Amatoriali
# Invocazione: /06-partite-amatoriali

Leggi le skill `@partite-amatoriali`, `@auth-ruoli`, `@react-patterns`, `@typescript-expert` e `@api-design-principles` prima di procedere.

Il tuo obiettivo è implementare il sistema completo delle partite amatoriali: registrazione a posteriori del risultato, doppia validazione del capitano avversario e assegnazione automatica dei punti.

> ⚠️ **Le partite si registrano DOPO aver giocato fisicamente.** Non c'è tracking live, nessun tabellone in tempo reale, nessuna partita "in corso". I giocatori giocano nella realtà, poi uno di loro apre l'app e inserisce il risultato. Non è necessario che i giocatori abbiano fatto check-in o siano sul campo.

> ⚠️ **I campi e le partite sono indipendenti.** Il `field_id` nella partita è solo informativo. La registrazione di una partita non aggiorna mai lo stato dei campi o delle presenze.

## Cosa devi produrre

### 1. Edge Function register-match
Crea `/supabase/functions/register-match/index.ts`. Riceve: `team_a_player2_id`, `team_b_player1_id`, `team_b_player2_id`, `field_id` (nullable), `sets` (array di `{score_a, score_b}`). Chi invoca la funzione è automaticamente `team_a_player1_id` e `captain_a`. Logica:
1. Determina se la partita è `ranked` o `unranked`: per ogni giocatore tra i 4, verifica se esiste già una partita ranked con `status IN ('pending_validation', 'disputed', 'completed')` registrata oggi (fuso `Europe/Rome`). Se tutti e 4 liberi → `ranked: true`; se anche uno solo ha già una ranked oggi → `ranked: false`.
2. Calcola il vincitore dai set inseriti (primo a 2 set vince)
3. Crea il record in `matches` con `status: 'pending_validation'`
4. Crea i record in `match_sets`
5. Invia notifica in-app al `captain_b` (primo giocatore del team avversario) per richiedere la conferma
Leggi `@partite-amatoriali` per la logica ranked e lo schema esatto.

### 2. Edge Function confirm-match-result
Crea `/supabase/functions/confirm-match-result/index.ts`. Può essere invocata solo dal `captain_b` della partita. Setta `opponent_captain_confirmed: true` e cambia `status: 'completed'`. Se la partita è `ranked: true` → chiama `assign-match-points`. Leggi `@partite-amatoriali` per i dettagli.

### 3. Edge Function dispute-match-result
Crea `/supabase/functions/dispute-match-result/index.ts`. Può essere invocata solo dal `captain_b`. Cambia `status: 'disputed'` e invia notifica a tutti gli utenti con ruolo `staff` o `admin` per richiedere risoluzione manuale. Leggi `@partite-amatoriali` per i casi edge.

### 4. Edge Function resolve-match-dispute
Crea `/supabase/functions/resolve-match-dispute/index.ts`. Può essere invocata solo da `staff` o `admin`. Riceve: `match_id`, `winner_team` ('a' o 'b') e i set corretti opzionali. Aggiorna il risultato, setta `status: 'completed'`. Se `ranked: true` → chiama `assign-match-points`. Usa `@auth-ruoli` per il controllo ruolo.

### 5. Edge Function assign-match-points
Crea `/supabase/functions/assign-match-points/index.ts`. Invocata solo internamente (da `confirm-match-result` e `resolve-match-dispute`). Se `ranked: true`: assegna +10 punti ai 2 giocatori del team vincitore aggiornando `points_balance` e `total_points_earned` in `profiles`, crea i record in `point_transactions` con `reason: 'match_win'`. Se `ranked: false`: non fa nulla. `point_transactions` non è mai scrivibile dalla UI. Leggi `@partite-amatoriali` per lo schema di `point_transactions`.

### 6. Cron Job timeout 24 ore (pg_cron)
Configura un job `pg_cron` in Supabase che ogni ora esegue: per ogni partita in `status: 'pending_validation'` con `created_at` di più di 24 ore fa → setta `status: 'completed'` e assegna i punti se `ranked: true`. Il timeout è di **24 ore**, non minuti — le partite si registrano a posteriori e non sono urgenti. Leggi `@partite-amatoriali` per la nota sul timeout.

### 7. Hook useMatchRegistration
Crea `src/hooks/useMatchRegistration.ts`. Incapsula la logica del form di registrazione partita. Espone: `registerMatch(data)`, `loading`, `error`, `ranked` (preview calcolata prima di confermare — chiama un endpoint o calcola in locale). Leggi `@partite-amatoriali` per la logica ranked che deve essere replicata anche lato client per mostrare il badge in anteprima.

### 8. Hook useMatches
Crea `src/hooks/useMatches.ts`. Carica lo storico partite dell'utente loggato (come giocatore in `team_a_ids` o `team_b_ids`). Espone: `matches` (con set e profili dei giocatori), `pendingValidation` (partite dove l'utente è `captain_b` e deve confermare), `loading`. Leggi `@api-design-principles` per la query con array contains e join.

### 9. Componente RegisterMatchModal
Crea `src/features/partite/RegisterMatchModal/index.tsx`. Modal multi-step per registrare una partita a posteriori:
- **Step 1 — Giocatori**: cerca e seleziona i 4 giocatori (2 per squadra). L'utente che registra è pre-selezionato come primo giocatore del suo team. Non è richiesto che i giocatori siano in struttura.
- **Step 2 — Campo (opzionale)**: selezione campo dove si è giocato, solo informativa.
- **Step 3 — Punteggio**: inserimento set per set (es. 21-18, 19-21, 15-12). Il sistema calcola automaticamente il vincitore.
- **Step 4 — Anteprima**: mostra badge prominente **RANKED ✅** o **UNRANKED ⚡** (determinato automaticamente), riepilogo giocatori e risultato. Bottone "Conferma e invia".
Leggi `@react-patterns` per modal multi-step e `@partite-amatoriali` per la logica ranked.

### 10. Componente PendingValidationBanner
Crea `src/features/partite/PendingValidationBanner/index.tsx`. Banner/card mostrato nella pagina Partite quando l'utente ha partite da confermare come `captain_b`. Mostra: le squadre, il risultato inserito, tipo (ranked/unranked), e due bottoni: **"✅ Confermo"** e **"❌ Contesto"**. Leggi `@partite-amatoriali` per i due flussi (conferma → `confirm-match-result`, contesta → `dispute-match-result`).

### 11. Componente MatchHistory
Crea `src/features/partite/MatchHistory/index.tsx`. Storico partite dell'utente con filtri (Tutte / Ranked / Unranked / In attesa). Per ogni partita mostra: badge ranked/unranked, data, squadre, risultato set per set, stato (completata / in attesa / in disputa), punti guadagnati. Leggi `@react-patterns` per liste filtrabili.

### 12. Componente DisputedMatchAdmin
Crea `src/features/partite/DisputedMatchAdmin/index.tsx`. Vista visibile solo a `staff` e `admin`. Lista delle partite in `status: 'disputed'`. Per ogni partita: mostra i dettagli e un form per inserire il risultato corretto e risolvere la disputa chiamando `resolve-match-dispute`. Usa `@auth-ruoli` per proteggere il componente per ruolo.

### 13. Pagina Partite
Crea `src/pages/Partite/index.tsx`. Struttura:
1. `PendingValidationBanner` in cima (solo se ci sono partite da confermare)
2. Bottone "Registra partita" che apre `RegisterMatchModal`
3. `MatchHistory` con storico completo
4. `DisputedMatchAdmin` in fondo (solo per staff/admin)
Proteggi la pagina con `useRequireAuth()`.

## Regole da rispettare

- **Nessun tracking live** — non creare stati `in_progress`, tabelloni o aggiornamenti in tempo reale per le partite (vedi `@partite-amatoriali`)
- I punti vengono assegnati **solo** dalla Edge Function `assign-match-points` — mai dalla UI
- `ranked` è determinato **automaticamente** dal sistema alla registrazione — l'utente non può sceglierlo
- La regola ranked controlla anche `pending_validation` e `disputed`, non solo `completed` — per evitare abusi
- Il timeout di validazione è **24 ore** (non minuti) — gestito con pg_cron lato server, non lato client
- Il `field_id` nella partita è solo informativo — **non aggiornare mai** lo stato dei campi o delle presenze dalla logica partite
- Usa il fuso `Europe/Rome` per determinare "oggi" nella regola ranked
- `point_transactions` mai esposto in insert/update dalla UI — solo Edge Functions con service_role
