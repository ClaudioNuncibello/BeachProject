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
Crea le tabelle `fields`, `presences`, `waiting_list` come descritte in `@campi-turni`. Abilita RLS su ognuna e scrivi le policy seguendo i pattern di `@auth-ruoli`.

### 4. Tabelle partite
Crea le tabelle `matches`, `match_sets`, `point_transactions` come descritte in `@partite-amatoriali`. Ricorda che `point_transactions` non deve essere mai scrivibile dalla UI — solo dalle Edge Functions. Scrivi le RLS policy di conseguenza.

### 5. Tabelle FantaBeach
Crea le tabelle `cards`, `user_cards`, `user_roster`, `tournaments`, `tournament_lineups`, `tournament_matches`, `tournament_results` come descritte in `@fanta-beach`. Scrivi le RLS policy per ogni tabella.

### 6. Indici
Per ogni tabella aggiungi indici sulle colonne usate frequentemente in WHERE e JOIN (es. `user_id`, `field_id`, `match_id`, `tournament_id`, `status`).

### 7. Bucket Storage
Aggiungi il comando per creare il bucket `carte-giocatori` in Supabase Storage, pubblico in lettura.

## Regole da rispettare

- Ogni tabella DEVE avere RLS abilitata — nessuna eccezione (vedi `@auth-ruoli`)
- Usa sempre `gen_random_uuid()` per le primary key
- Usa `timestamptz` per tutti i timestamp, mai `timestamp`
- Aggiungi `created_at timestamptz default now()` su ogni tabella
- I check constraint devono coprire tutti i valori enum (es. `status`, `role`, `rarity`)
- Scrivi un commento in italiano sopra ogni sezione che spiega cosa fa
