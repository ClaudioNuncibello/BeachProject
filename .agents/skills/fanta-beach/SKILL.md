---
name: fanta-beach
description: Skill per tutto il sistema FantaBeach — carte collezionabili, shop, rosa, eventi FantaBeach e classifica. Usa quando lavori su qualsiasi feature della sezione FantaBeach. NOTA: FantaBeach è un gioco virtuale completamente separato dai tornei fisici.
---

# Skill: FantaBeach

> [!CAUTION]
> **🔜 COMING SOON — Non implementare nella v1.**
> Questa skill documenta la visione futura del FantaBeach per riferimento e progettazione. Nella v1 la sezione FantaBeach è un placeholder "Coming Soon" nella navigazione. Non scrivere codice per queste feature finché non viene esplicitamente rimosso questo avviso.

## Visione generale

Il FantaBeach è un gioco **virtuale** stile fantacalcio applicato al beach volley locale. È completamente separato dai tornei fisici organizzati in struttura.

> [!IMPORTANT]
> **FantaBeach NON è collegato ai tornei fisici.** I tornei fisici sono eventi reali gestiti dai moderatori nella sezione "Tornei". Il FantaBeach ha i propri "eventi virtuali" creati dagli admin, con logica di punteggio completamente indipendente.

Il ciclo del FantaBeach:
1. L'utente guadagna punti (da partite ranked e tornei fisici)
2. Compra carte dei giocatori della struttura con i punti
3. Crea una rosa di 3 coppie (6 giocatori)
4. Quando un admin crea un **Evento FantaBeach**, schiera una delle 3 coppie
5. L'admin assegna manualmente i punti FantaBeach agli utenti dopo l'evento, basandosi su competizioni live esterne all'app

## Sistema Carte

### Struttura carta
- Ogni carta rappresenta un giocatore reale della struttura
- Le grafiche sono custom (prodotte da un gruppo di amici, ancora in concept)
- Ogni carta ha: `player_name`, `image_url`, `rarity` (comune/raro/leggendario), `cost` (punti), `stats` (opzionale v1)
- Le carte sono acquistabili nello **shop** con i `points_balance`

### Collezione
- Ogni utente ha un inventario di carte (`user_cards`)
- Si possono avere più copie della stessa carta
- La UI mostra le carte come griglia di figurine con immagine e nome

### Shop
- Lista di tutte le carte disponibili con costo in punti
- L'utente può comprare solo se `points_balance >= card.cost`
- L'acquisto scala `points_balance` ma NON `total_points_earned`

## Sistema Rosa

- Ogni utente crea **3 coppie** dalla propria collezione
- Una coppia = 2 giocatori (2 carte) dalla propria collezione
- Un giocatore non può essere in due coppie contemporaneamente
- Le coppie possono essere rinominate (es. "La Corazzata")
- La rosa è modificabile liberamente fuori dal periodo di schieramento di un evento

## Eventi FantaBeach

> [!IMPORTANT]
> Gli eventi FantaBeach sono creati manualmente dagli admin. Non sono automatici e non sono collegati ai tornei fisici. I punti vengono assegnati manualmente dagli admin dopo l'evento, basandosi su competizioni live esterne all'app.

### Ciclo di vita di un Evento FantaBeach
1. **Creazione** (admin): nome evento, data, finestra di schieramento
2. **Schieramento** (utenti): ogni utente sceglie quale delle 3 coppie schierare entro la scadenza
3. **Svolgimento**: l'evento si svolge nel mondo reale, fuori dall'app
4. **Assegnazione punti** (admin): dopo l'evento, l'admin inserisce manualmente i punti per ogni utente partecipante
5. **Chiusura**: l'evento viene marcato come completato

### Schieramento
- Prima della scadenza, ogni utente sceglie **una delle 3 coppie** da schierare
- Finché la finestra non chiude, lo schieramento è modificabile
- L'utente che non schiera prima della scadenza non partecipa all'evento

### Punti evento FantaBeach
- I punti sono assegnati **manualmente dall'admin** dopo l'evento
- Non c'è una scala fissa: l'admin decide i punti per ogni partecipante in base alle performance reali
- I punti vengono aggiunti a `points_balance` e `total_points_earned`
- L'assegnazione avviene tramite una schermata admin nell'app (non via Edge Function automatica)

## Classifica FantaBeach

- Classifica globale ordinata per `total_points_earned` DESC
- Mostra: posizione, avatar, nome, punti totali guadagnati
- Il saldo (`points_balance`) è privato, visibile solo nel proprio profilo

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

-- Eventi FantaBeach (creati dagli admin, distinti dai tornei fisici)
fanta_events (
  id uuid,
  name text,
  description text nullable,
  event_date date,
  lineup_deadline timestamptz,   -- scadenza per lo schieramento
  status text check (status in ('upcoming', 'lineup_open', 'in_progress', 'completed')),
  created_by uuid references profiles(id)
)

-- Schieramenti utenti per evento FantaBeach
fanta_event_lineups (
  id uuid,
  fanta_event_id uuid references fanta_events(id),
  user_id uuid references profiles(id),
  roster_slot int check (roster_slot in (1, 2, 3)),
  submitted_at timestamptz,
  unique(fanta_event_id, user_id)
)

-- Punti assegnati dagli admin post-evento
fanta_event_results (
  id uuid,
  fanta_event_id uuid references fanta_events(id),
  user_id uuid references profiles(id),
  points_awarded int,
  notes text nullable,        -- note facoltative dell'admin sul perché quei punti
  awarded_by uuid references profiles(id),
  awarded_at timestamptz
)

-- Transazioni punti (condivisa con partite e tornei)
point_transactions (
  id uuid,
  user_id uuid references profiles(id),
  amount int,
  reason text,    -- 'match_win', 'tournament_1st', 'fanta_event', ecc.
  reference_id uuid,
  created_at timestamptz
)
```

## RLS da implementare

- `cards`: tutti possono leggere, solo admin possono inserire/modificare
- `user_cards`: ogni utente vede solo le proprie, inserimento solo via Edge Function (acquisto)
- `user_roster`: ogni utente legge e modifica solo la propria
- `fanta_events`: tutti possono leggere, solo admin possono creare/modificare
- `fanta_event_lineups`: ogni utente legge e modifica la propria (se entro la deadline)
- `fanta_event_results`: tutti possono leggere, solo admin possono inserire/modificare

## Edge Functions / operazioni admin necessarie

- `purchase-card` — acquisto carta: verifica saldo, scala `points_balance`, crea `user_cards`
- `close-fanta-event-lineups` — blocca gli schieramenti alla deadline
- `award-fanta-event-points` — (chiamata da UI admin) assegna punti a un utente per un evento, crea `point_transactions`, aggiorna `points_balance` e `total_points_earned`

## Componenti UI principali

1. **CardGrid** — griglia carte con immagine, nome, rarità (stile album figurine)
2. **ShopPage** — catalogo carte acquistabili con saldo in evidenza
3. **RosterEditor** — editor delle 3 coppie con drag & drop delle carte
4. **FantaEventList** — lista eventi FantaBeach (upcoming / in corso / completati)
5. **LineupSelector** — scelta coppia da schierare per un evento (entro la deadline)
6. **FantaEventAdminPanel** — schermata admin per assegnare punti post-evento
7. **Leaderboard** — classifica globale FantaBeach per `total_points_earned`

## Note implementative

- La schermata admin `FantaEventAdminPanel` mostra tutti gli utenti che hanno schierato per quell'evento, con un campo punti modificabile per ciascuno
- L'assegnazione punti è **bulk** (l'admin compila tutti i campi e salva in una sola operazione) per ridurre gli errori
- Usare Realtime per aggiornare la classifica in tempo reale dopo l'assegnazione
- Non ci sono calcoli automatici sui punti FantaBeach — l'admin ha pieno controllo
