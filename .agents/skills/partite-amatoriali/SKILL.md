---
name: partite-amatoriali
description: Skill per la gestione delle partite amatoriali. Le partite si registrano a posteriori — prima si gioca fisicamente, poi si inserisce il risultato nell'app. Completamente separate dalla gestione dei campi e dalle presenze.
---

# Skill: Partite Amatoriali

> [!IMPORTANT]
> **Le partite sono concettualmente separate dalla gestione dei campi.** Un utente può essere in campo senza creare una partita. La partita si crea solo quando tutti e 4 i giocatori decidono esplicitamente. Il `field_id` nella partita è **solo informativo** e non aggiorna lo stato del campo.

## Logica di business fondamentale

> [!IMPORTANT]
> **Le partite si registrano a posteriori.** I giocatori giocano fisicamente, poi uno di loro apre l'app e registra il risultato. Non c'è tracking live, non c'è connessione con i campi o con le presenze.

### Registrazione partita
- Chiunque dei 4 giocatori può aprire l'app e registrare la partita dopo averla giocata
- Si selezionano i 4 giocatori (2 coppie), il campo dove si è giocato (opzionale, solo informativo), e il punteggio finale per set
- Chi registra la partita diventa automaticamente "capitano" della propria coppia
- L'altro capitano è il primo giocatore dell'altra coppia indicato
- Non è necessario che i giocatori siano in struttura o abbiano fatto check-in
- La partita viene creata in stato `pending_validation` — deve essere confermata da entrambi i capitani

### Regola Ranked / Unranked

> [!IMPORTANT]
> **Ogni utente può avere al massimo 1 partita "ranked" al giorno.** La regola si applica a livello di singolo utente.

- Quando viene creata una partita, il sistema verifica se **uno qualsiasi dei 4 giocatori** ha già una partita ranked nella giornata corrente
- Se tutti e 4 non hanno ancora una ranked oggi → la partita è `ranked: true`
- Se almeno uno dei 4 ha già una ranked oggi → la partita è `ranked: false` (unranked)
- L'utente viene informato al momento della creazione se la partita sarà ranked o unranked
- Non è possibile scegliere manualmente: è determinato automaticamente dal sistema
- Le partite **unranked** si giocano normalmente con tracking e validazione, ma non generano punti

### Inserimento punteggio
- Il punteggio viene inserito direttamente al momento della registrazione (non live)
- Si inserisce il risultato set per set (es. 21-18, 19-21, 15-12)
- Il sistema calcola automaticamente il vincitore in base ai set vinti
- Default: primo a 2 set vince, ogni set a 21 punti

### Doppia validazione
- La partita viene creata in `pending_validation` — il risultato deve essere confermato
- **Il capitano avversario** (non il registrante) riceve una notifica in-app per confermare
- Se il capitano avversario conferma → partita `completed`, punti assegnati
- Se il capitano avversario contesta → stato `disputed` → notifica admin/staff per risoluzione manuale
- Timeout: se il capitano avversario non risponde entro **24 ore**, il risultato viene accettato automaticamente (partita a posteriori, non urgente)

### Punti post-partita
- **Solo se `ranked: true`**: Vincitori +10 pt (`points_balance` e `total_points_earned`)
- Perdenti: 0 punti (nessuna penalità)
- Se `ranked: false`: nessun punto assegnato

## Schema DB rilevante

```sql
-- Partite
matches (
  id uuid,
  field_id uuid references fields(id) nullable,  -- solo informativo
  team_a_ids uuid[],
  team_b_ids uuid[],
  captain_a_id uuid,
  captain_b_id uuid,
  ranked bool default true,   -- determinato automaticamente alla creazione
  status text check (status in (
    'pending_validation',     -- appena registrata, in attesa conferma capitano avversario
    'disputed',               -- risultato contestato, in attesa risoluzione staff
    'completed'               -- validata e chiusa
  )),
  winner_team text check (winner_team in ('a', 'b')) nullable,
  registered_by uuid references profiles(id),  -- chi ha registrato la partita
  opponent_captain_confirmed bool default false,  -- solo il capitano avversario deve confermare
  created_at timestamptz,
  completed_at timestamptz nullable
)

-- Set della partita
match_sets (
  id uuid,
  match_id uuid references matches(id),
  set_number int,
  score_a int default 0,
  score_b int default 0,
  winner text check (winner in ('a', 'b')) nullable,
  completed_at timestamptz nullable
)

-- Transazioni punti (condivisa con tornei e FantaBeach)
point_transactions (
  id uuid,
  user_id uuid references profiles(id),
  amount int,
  reason text,    -- 'match_win', 'tournament_1st', 'fanta_event', ecc.
  reference_id uuid,
  created_at timestamptz
)
```

## Logica ranked (Edge Function)

```
La data di riferimento è quella del giorno in cui viene registrata la partita (Europe/Rome).

Per ogni playerId in [team_a + team_b]:
  se esiste una ranked con status IN ('pending_validation', 'disputed', 'completed')
  registrata oggi (fuso Europe/Rome) → partita = unranked
Se tutti liberi → partita = ranked
```

NOTA: si controlla anche `pending_validation` e `disputed` (non solo `completed`) per evitare che
un utente registri più partite ranked nello stesso giorno sfruttando il tempo di validazione.

## RLS da implementare

- `matches`: tutti possono leggere, solo i partecipanti possono aggiornare
- `match_sets`: tutti possono leggere, solo i partecipanti della partita possono aggiornare
- `point_transactions`: ogni utente legge solo le proprie, inserimento solo via Edge Functions

## Edge Functions necessarie

- `register-match` — registra la partita a posteriori, determina `ranked`, notifica il capitano avversario
- `confirm-match-result` — il capitano avversario conferma il risultato → `completed` + punti
- `dispute-match-result` — il capitano avversario contesta → `disputed` + notifica staff
- `resolve-match-dispute` — staff risolve manualmente la disputa
- `assign-match-points` — assegna punti solo se `ranked = true`

## Componenti UI principali

1. **RegisterMatchModal** — selezione avversari, campo (opzionale), inserimento punteggio set per set; mostra anteprima ranked/unranked prima della conferma
2. **PendingValidationBanner** — notifica al capitano avversario con pulsanti "Confermo" / "Contesto"
3. **MatchHistory** — storico partite con tipo (ranked/unranked), risultato e punti guadagnati
4. **DisputedMatchAdmin** — vista staff per risolvere le partite in dispute

## Note implementative

- Non c'è tracking live — la partita è un form di inserimento dati, non una sessione in tempo reale
- Usare `useMatchRegistration` hook per la logica di registrazione
- Badge ranked/unranked mostrato chiaramente nel modal prima della conferma
- Usare il fuso `Europe/Rome` per determinare "oggi" nella regola ranked
- `point_transactions` mai esposto in insert/update dalla UI — solo Edge Functions
- Il timeout di 24 ore per la validazione può essere gestito da un cron job Supabase (pg_cron)
