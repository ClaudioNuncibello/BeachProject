---
name: fanta-beach
description: Skill per tutto il sistema FantaBeach — carte collezionabili, shop, rosa, schieramento pre-torneo, gestione tornei ufficiali e classifica. Usa quando lavori su qualsiasi feature della sezione FantaBeach.
---

# Skill: FantaBeach

## Visione generale

Il FantaBeach è un gioco di fantacalcio applicato al beach volley locale. Gli utenti:
1. Guadagnano punti vincendo partite e tornei
2. Comprano carte dei giocatori della struttura con i punti
3. Creano una rosa di 3 coppie (6 giocatori)
4. Scelgono quale coppia schierare prima di ogni torneo ufficiale
5. Guadagnano punti FantaBeach in base al piazzamento della coppia schierata

## Sistema Carte

### Struttura carta
- Ogni carta rappresenta un giocatore reale della struttura
- Le grafiche sono custom (prodotte da un gruppo di amici, ancora in concept)
- Una carta ha: `player_name`, `image_url`, `rarity` (comune/raro/leggendario), `cost` (punti per acquistarla), `stats` (opzionale nella v1)
- Le carte sono acquistabili nello **shop** con i `points_balance`

### Collezione
- Ogni utente ha un inventario di carte (`user_cards`)
- Si possono avere più copie della stessa carta
- La UI mostra le carte come una griglia di figurine con immagine e nome

### Shop
- Lista di tutte le carte disponibili con costo in punti
- L'utente può comprare solo se `points_balance >= card.cost`
- L'acquisto scala `points_balance` ma NON `total_points_earned`

## Sistema Rosa

- Ogni utente può creare **3 coppie** dalla propria collezione
- Una coppia = 2 giocatori (2 carte) dalla propria collezione
- Un giocatore non può essere in due coppie contemporaneamente
- Le coppie possono essere rinominate (es. "La Corazzata", "Team Catania")
- La rosa è modificabile liberamente fuori dal periodo di schieramento torneo

## Tornei Ufficiali

### Ciclo di vita torneo (gestito solo da Staff/Admin)
1. **Creazione**: nome, data, formato (eliminazione diretta / girone)
2. **Apertura schieramenti**: gli utenti scelgono quale coppia schierare (finestra temporale)
3. **Svolgimento**: l'arbitro aggiorna il bracket in real-time durante l'evento fisico
4. **Chiusura**: admin conferma i piazzamenti finali
5. **Assegnazione punti**: automatica via Edge Function

### Schieramento
- Prima di ogni torneo, ogni utente sceglie **una delle 3 coppie** da schierare
- La coppia schierata deve essere composta da giocatori che partecipano fisicamente al torneo
- Finché il torneo non inizia, lo schieramento è modificabile

### Punti torneo FantaBeach
I punti guadagnati dall'utente dipendono dal piazzamento della **coppia schierata**:
- 1° posto → 100 pt
- 2° posto → 70 pt
- 3° posto → 50 pt
- Partecipazione → 20 pt

### Bracket real-time
- Il bracket è visibile a tutti gli utenti durante il torneo
- Solo Staff/Admin possono aggiornare i risultati delle partite del torneo
- Usare Supabase Realtime per aggiornamenti live del tabellone

## Classifica FantaBeach

- Classifica globale di tutti gli utenti ordinata per `total_points_earned` DESC
- Mostra: posizione, avatar, nome, punti totali guadagnati
- Non mostra il saldo (quello è privato e visibile solo nel proprio profilo)

## Schema DB rilevante

```sql
-- Catalogo carte
cards (
  id uuid,
  player_name text,
  image_url text,
  rarity text check (rarity in ('common', 'rare', 'legendary')),
  cost int,
  is_available bool default true
)

-- Carte possedute dagli utenti
user_cards (
  id uuid,
  user_id uuid references profiles(id),
  card_id uuid references cards(id),
  acquired_at timestamptz
)

-- Rosa utente (3 coppie)
user_roster (
  id uuid,
  user_id uuid references profiles(id),
  couple_slot int check (couple_slot in (1, 2, 3)),
  card_a_id uuid references user_cards(id) nullable,
  card_b_id uuid references user_cards(id) nullable,
  couple_name text nullable
)

-- Tornei
tournaments (
  id uuid,
  name text,
  date date,
  format text check (format in ('elimination', 'round_robin')),
  status text check (status in ('upcoming', 'registration_open', 'in_progress', 'completed')),
  created_by uuid references profiles(id)
)

-- Schieramenti utenti per torneo
tournament_lineups (
  id uuid,
  tournament_id uuid references tournaments(id),
  user_id uuid references profiles(id),
  roster_slot int check (roster_slot in (1, 2, 3)),  -- quale delle 3 coppie
  locked bool default false,
  unique(tournament_id, user_id)
)

-- Partite del torneo (bracket)
tournament_matches (
  id uuid,
  tournament_id uuid references tournaments(id),
  round int,
  team_a text,   -- nome coppia reale
  team_b text,
  score_a int nullable,
  score_b int nullable,
  winner text nullable,
  status text check (status in ('scheduled', 'in_progress', 'completed'))
)

-- Piazzamenti finali
tournament_results (
  id uuid,
  tournament_id uuid references tournaments(id),
  team_name text,
  position int
)
```

## RLS da implementare

- `cards`: tutti possono leggere, solo admin possono inserire/modificare
- `user_cards`: ogni utente può vedere solo le proprie, inserimento solo via Edge Function (acquisto)
- `user_roster`: ogni utente può leggere e modificare solo la propria
- `tournament_lineups`: ogni utente può leggere e modificare la propria (se torneo non locked)
- `tournaments`: tutti possono leggere, solo staff/admin possono creare/modificare
- `tournament_matches`: tutti possono leggere, solo staff/admin possono aggiornare

## Edge Functions necessarie

- `purchase-card` — acquisto carta: verifica saldo, scala `points_balance`, crea `user_cards`
- `assign-tournament-points` — dopo chiusura torneo: legge piazzamenti, assegna punti a chi ha schierato quella coppia
- `lock-tournament-lineups` — blocca gli schieramenti all'inizio del torneo

## Componenti UI principali

1. **CardGrid** — griglia carte con immagine, nome, rarità (stile album figurine)
2. **ShopPage** — catalogo carte acquistabili con saldo in evidenza
3. **RosterEditor** — editor delle 3 coppie con drag & drop delle carte
4. **TournamentBracket** — tabellone torneo real-time
5. **LineupSelector** — scelta coppia da schierare prima del torneo
6. **Leaderboard** — classifica globale FantaBeach
