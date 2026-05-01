# Workflow: Schema DB Completo
# Invocazione: /01-schema-db

Leggi le skill `@project-context`, `@auth-ruoli`, `@campi-turni`, `@partite-amatoriali`, `@fanta-beach` e `@api-design-principles` prima di procedere.

Il tuo obiettivo è creare il file `/supabase/migrations/00000000000000_initial_schema.sql` che contiene l'intero schema del database dell'app Beach Volley Catania.

## Cosa devi produrre

Crea un unico file SQL con questo ordine:

### 1. Tabella profiles
Estende `auth.users` di Supabase. Ogni utente ha: `id`, `username`, `avatar_url`, `role` (default `player`), `points_balance` (default 0), `total_points_earned` (default 0). Leggi `@auth-ruoli` per la struttura esatta e il trigger `handle_new_user` che crea automaticamente il profilo alla registrazione.

### 2. Funzioni helper ruolo
Crea le funzioni `get_user_role()`, `is_staff_or_above()`, `is_admin()` come descritte in `@auth-ruoli`. Queste funzioni verranno usate in tutte le RLS policy del progetto.

### 3. Tabelle campi e presenze
Crea le tabelle `fields`, `presences`, `field_sessions`, `waiting_list` come descritte in `@campi-turni`. Rispetta questi dettagli:
- `fields.status` → valori: `'free'`, `'partial'`, `'occupied'`, `'maintenance'`
- `fields.couples_on_field` → int tra 0 e 2
- `field_sessions`: una riga per coppia attiva su un campo (max 2 per campo); `ended_at` null = sessione in corso
- `waiting_list.position` → int ordinato per `created_at` FIFO
- In `field_sessions` e `waiting_list`: `player_2_id` e `player_2_guest_name` sono mutualmente esclusivi — aggiungi un CHECK constraint
- La tabella `presences` NON ha `field_id` — il campo dell'utente si ricava con JOIN su `field_sessions`
Abilita RLS su ognuna e scrivi le policy seguendo i pattern di `@auth-ruoli` e le specifiche di `@campi-turni`.

### 4. Tabelle partite
Crea le tabelle `matches`, `match_sets`, `point_transactions` come descritte in `@partite-amatoriali`. Dettagli:
- `matches.status` → valori: `'pending_validation'`, `'disputed'`, `'completed'`
- `matches.field_id` è nullable e solo informativo — non ha effetti sullo stato del campo
- `matches.ranked` → bool determinato automaticamente alla creazione (non scelto dall'utente)
- `point_transactions` non deve mai essere scrivibile dalla UI — solo dalle Edge Functions
Scrivi le RLS policy di conseguenza.

### 5. Tabelle FantaBeach
> ⚠️ FantaBeach è Coming Soon v2+. Crea le tabelle per riferimento futuro ma **non implementare le feature nella v1**.

Crea le tabelle: `cards`, `user_cards`, `user_roster`, `fanta_events`, `fanta_event_lineups`, `fanta_event_results` come descritte in `@fanta-beach`. Dettagli:
- `fanta_events.status` → valori: `'upcoming'`, `'lineup_open'`, `'in_progress'`, `'completed'`
- `fanta_event_lineups` ha `unique(fanta_event_id, user_id)`
- `fanta_event_results.points_awarded` è assegnato manualmente dagli admin — no calcolo automatico
- NON creare tabelle per i tornei fisici (Tornei = Coming Soon separato, da progettare in futuro)
Scrivi le RLS policy per ogni tabella.

### 6. Indici
Per ogni tabella aggiungi indici sulle colonne usate frequentemente in WHERE e JOIN (es. `user_id`, `field_id`, `match_id`, `fanta_event_id`, `status`, `created_at`).

### 7. Bucket Storage
Aggiungi il comando per creare il bucket `carte-giocatori` in Supabase Storage, pubblico in lettura.

## Regole da rispettare

- Ogni tabella DEVE avere RLS abilitata — nessuna eccezione (vedi `@auth-ruoli`)
- Usa sempre `gen_random_uuid()` per le primary key
- Usa `timestamptz` per tutti i timestamp, mai `timestamp`
- Aggiungi `created_at timestamptz default now()` su ogni tabella
- I check constraint devono coprire tutti i valori enum (es. `status`, `role`, `rarity`)
- Scrivi un commento in italiano sopra ogni sezione che spiega cosa fa
- `presences` non ha `field_id` — usare sempre JOIN con `field_sessions` per sapere dove si trova un utente (vedi `@campi-turni`)
