---
name: partite-amatoriali
description: Skill per la gestione delle partite amatoriali create dai giocatori. Usa quando lavori sulla creazione partita, tracking set, doppia validazione del risultato, o assegnazione punti post-partita.
---

# Skill: Partite Amatoriali

## Logica di business fondamentale

### Creazione partita
- Solo un utente con `status: 'available'` può creare una partita
- La partita richiede: campo, 2 coppie (4 giocatori totali presenti in struttura)
- Chi crea la partita diventa automaticamente "capitano" della propria coppia
- L'altro capitano è il primo giocatore dell'altra coppia
- La partita inizia solo dopo che **entrambi i capitani** hanno confermato la partecipazione
- Alla conferma: i 4 giocatori passano a `status: 'playing'`, il campo passa a `playing`

### Tracking set
- Una partita è composta da set (di default: primo a 2 set vince, ogni set a 21 punti)
- I punti del set sono aggiornabili da **qualsiasi giocatore delle due coppie** (non solo i capitani)
- Lo stato del set è in Realtime — tutti i giocatori vedono l'aggiornamento live

### Risultato e doppia validazione
- Quando una coppia raggiunge i punti necessari per vincere l'ultimo set, la partita entra in stato `pending_validation`
- Il sistema identifica automaticamente la coppia vincitrice in base ai set vinti
- **Entrambi i capitani** devono confermare il risultato
- Se c'è disaccordo → stato `disputed` → notifica admin/staff per risoluzione manuale
- Timeout: se un capitano non risponde entro 10 minuti, il risultato viene accettato automaticamente

### Punti post-partita
- Alla validazione confermata → Edge Function `assign-match-points` viene invocata
- Vincitori: +10 punti (`points_balance` e `total_points_earned`)
- Perdenti: 0 punti (le partite amatoriali non penalizzano)
- Il campo torna `free`, i giocatori tornano `available`, il primo in lista d'attesa viene notificato

## Schema DB rilevante

```sql
-- Partite
matches (
  id uuid,
  field_id uuid references fields(id),
  team_a_ids uuid[],          -- array di 2 user_id
  team_b_ids uuid[],          -- array di 2 user_id
  captain_a_id uuid,
  captain_b_id uuid,
  status text check (status in (
    'pending_start',          -- creata, in attesa conferma capitani
    'in_progress',            -- in corso
    'pending_validation',     -- finita, in attesa conferma risultato
    'disputed',               -- risultato contestato
    'completed'               -- validata e chiusa
  )),
  winner_team text check (winner_team in ('a', 'b')) nullable,
  captain_a_confirmed bool default false,
  captain_b_confirmed bool default false,
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

-- Transazioni punti
point_transactions (
  id uuid,
  user_id uuid references profiles(id),
  amount int,
  reason text,               -- es. 'match_win', 'tournament_1st'
  reference_id uuid,         -- match_id o tournament_id
  created_at timestamptz
)
```

## RLS da implementare

- `matches`: tutti possono leggere, solo i partecipanti possono aggiornare il proprio set
- `match_sets`: tutti possono leggere, solo i partecipanti della partita possono aggiornare
- `point_transactions`: ogni utente può leggere solo le proprie, nessuno può inserire direttamente (solo Edge Functions)

## Edge Functions necessarie

- `create-match` — crea la partita e notifica i capitani
- `confirm-match-start` — entrambi i capitani confermano, match parte
- `validate-match-result` — gestisce la doppia validazione e i casi di dispute
- `assign-match-points` — assegna i punti, aggiorna campo e lista d'attesa

## Componenti UI principali

1. **CreateMatchModal** — selezione campo, avversari, conferma
2. **MatchScoreboard** — tabellone live con punteggio set, aggiornabile dai giocatori
3. **ValidationConfirm** — dialog per confermare il risultato finale (mostrato ai capitani)
4. **MatchHistory** — storico partite dell'utente con risultati e punti guadagnati

## Note implementative

- Usare `useMatch` hook per tutta la logica della partita corrente
- Il tabellone deve essere **Realtime** — subscription su `match_sets` per aggiornamenti live
- La validazione usa un pattern ottimistico: aggiorna UI subito, poi sincronizza con DB
- Non esporre mai `point_transactions` in insert/update dalla UI — solo le Edge Functions possono scrivere su quella tabella
