---
name: campi-turni
description: Skill per tutto ciò che riguarda la gestione dei campi, il check-in degli utenti, la lista d'attesa e la rotazione dei turni. Usa quando stai lavorando su componenti come la mappa campi, il sistema di presenza, o la logica di coda.
---

# Skill: Campi & Turni

## Logica di business fondamentale

### Check-in
- Un utente fa check-in all'arrivo in struttura
- Il check-in crea un record `presences` con `status: 'available'`
- Ogni utente può avere un solo check-in attivo alla volta
- Il check-out (manuale o automatico a mezzanotte) setta `status: 'checked_out'`

### Stato dei campi
- Ogni campo ha uno stato: `free` | `playing` | `maintenance`
- Quando una partita inizia su un campo → campo diventa `playing`
- Quando la partita finisce → campo torna `free`
- Lo stato dei campi è in **Supabase Realtime** — usare subscription, non polling

### Lista d'attesa e rotazione
- Se un campo è occupato, i nuovi arrivati si aggiungono alla coda (`waiting_list`)
- La coda è ordinata per `created_at` (FIFO)
- Regola di rotazione: **vincitore resta, perdente lascia il campo**
- Quando una partita finisce, il sistema notifica automaticamente il primo in lista
- Il primo in lista ha X minuti per accettare prima di perdere il posto (da definire, suggerito: 3 minuti)

## Schema DB rilevante

```sql
-- Presenze giornaliere
presences (
  id uuid,
  user_id uuid references profiles(id),
  field_id uuid references fields(id) nullable,
  status text check (status in ('available', 'playing', 'waiting', 'checked_out')),
  checked_in_at timestamptz,
  checked_out_at timestamptz
)

-- Campi della struttura
fields (
  id uuid,
  name text,           -- es. "Campo 1", "Campo 2"
  status text check (status in ('free', 'playing', 'maintenance')),
  current_match_id uuid nullable
)

-- Lista d'attesa per campo
waiting_list (
  id uuid,
  field_id uuid references fields(id),
  presence_id uuid references presences(id),
  position int,
  created_at timestamptz,
  notified_at timestamptz nullable
)
```

## RLS da implementare

- `presences`: ogni utente può vedere tutte le presenze (per sapere chi c'è), ma può modificare solo la propria
- `fields`: tutti possono leggere, solo staff/admin possono modificare lo stato
- `waiting_list`: tutti possono leggere, ogni utente può aggiungere/rimuovere solo se stesso

## Componenti UI principali

1. **FieldMap** — griglia dei campi con stato live, mostra chi gioca
2. **CheckInButton** — pulsante toggle check-in/check-out nella home
3. **WaitingQueue** — lista d'attesa per campo con posizione dell'utente evidenziata
4. **AttendanceList** — lista di chi è in struttura oggi (nome + stato)

## Note implementative

- Usare `usePresence` hook per incapsulare tutta la logica di check-in
- Usare `useFieldStatus` hook con subscription Realtime per lo stato dei campi
- La notifica "sei il prossimo" può essere un toast in-app (non push notification nella v1)
- Il cambio di turno deve essere **atomico** — usare una Supabase Edge Function `handle-match-end` che aggiorna campo + waiting list in una transazione
