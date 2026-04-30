---
name: campi-turni
description: Skill per tutto ciò che riguarda la gestione dei campi, il check-in degli utenti, la lista d'attesa e la rotazione dei turni. Usa quando stai lavorando su componenti come la mappa campi, il sistema di presenza, o la logica di coda. NOTA: i campi sono indipendenti dalle partite amatoriali.
---

# Skill: Campi & Turni

> [!IMPORTANT]
> **I campi sono concettualmente separati dalle partite.** Occupare un campo non implica la creazione di una partita nell'app. Le partite si registrano a posteriori in una sezione separata.

## Logica di business fondamentale

### Check-in
- Un utente fa check-in all'arrivo in struttura
- Il check-in crea un record `presences` con `status: 'available'`
- Ogni utente può avere un solo check-in attivo alla volta
- Il check-out (manuale o automatico a mezzanotte) setta `status: 'checked_out'`

### Struttura dei campi — coppie e slot

Un campo può ospitare **al massimo 2 coppie contemporaneamente** (4 giocatori totali). La coda gestisce l'ingresso **una coppia alla volta**.

| Coppie sul campo | Stato campo | Comportamento |
|---|---|---|
| 0 | `free` | Il primo in coda entra automaticamente |
| 1 | `partial` | Il secondo in coda entra automaticamente; campo diventa `occupied` |
| 2 | `occupied` | Le coppie successive aspettano in coda |

> [!IMPORTANT]
> **Auto-fill**: se il campo ha meno di 2 coppie e c'è almeno una coppia in coda, quella coppia entra automaticamente senza dover accettare manualmente. L'auto-fill avviene anche al momento dell'aggiunta alla coda (se c'è posto, si entra subito).

### Composizione di una coppia in coda

- La coppia è formata da **2 giocatori**
- Il **primo giocatore** deve essere un utente registrato (chi crea la voce in coda)
- Il **secondo giocatore** può essere:
  - Un altro utente registrato (selezionato dall'app)
  - Un **ospite** senza profilo app — si inserisce solo il nome come testo libero

### Rimozione dal campo — modello comunitario

> [!IMPORTANT]
> **Chiunque può rimuovere una coppia dal campo.** Non solo i diretti interessati. Questo accelera la rotazione: quando una coppia finisce di giocare, qualsiasi utente può segnalarlo nell'app. All'uscita di una coppia, il sistema promuove automaticamente la prima coppia in coda.

La rimozione è un'azione sociale/comunitaria, non richiede permessi speciali. L'abuso è improbabile nel contesto di una struttura con giocatori abituali.

### Lista d'attesa e rotazione

- Se il campo è `occupied`, le nuove coppie si aggiungono alla coda (`waiting_list`)
- La coda è ordinata per `created_at` (FIFO)
- Quando una coppia viene rimossa dal campo:
  1. La `field_session` viene chiusa (`ended_at = now()`)
  2. Se ci sono coppie in coda → la prima entra automaticamente (nuova `field_session`)
  3. Lo stato del campo viene aggiornato di conseguenza
- Non c'è un timer di accettazione — l'auto-fill è immediato e non richiede conferma

### Durata turno
- Nessun limite di tempo fisso nella v1 (la struttura gestisce informalmente)
- In futuro si potrà aggiungere un timer opzionale per turno

## Schema DB rilevante

```sql
-- Presenze giornaliere
presences (
  id uuid,
  user_id uuid references profiles(id),
  -- NOTA: nessun field_id qui. Il campo dell'utente si ricava con JOIN su field_sessions.
  -- presences traccia SOLO lo status; field_sessions è la source of truth per la posizione.
  status text check (status in ('available', 'on_field', 'waiting', 'checked_out')),
  checked_in_at timestamptz,
  checked_out_at timestamptz
)
```

> [!TIP]
> **Pattern consigliato per "chi è su quale campo"**: fare una LEFT JOIN tra `presences` e `field_sessions` (filtrando su `ended_at IS NULL`). Non aggiungere `field_id` in `presences` — sarebbe dato duplicato soggetto a inconsistenza.
>
> ```sql
> select p.user_id, p.status, fs.field_id
> from presences p
> left join field_sessions fs
>   on (p.user_id = fs.player_1_id or p.user_id = fs.player_2_id)
>   and fs.ended_at is null
> where p.status = 'on_field';
> ```

```sql

-- Campi della struttura
fields (
  id uuid,
  name text,           -- es. "Campo 1", "Campo 2"
  status text check (status in ('free', 'partial', 'occupied', 'maintenance')),
  -- free = 0 coppie, partial = 1 coppia, occupied = 2 coppie, maintenance = fuori uso
  couples_on_field int default 0 check (couples_on_field between 0 and 2)
)

-- Sessioni attive sul campo (una per coppia; max 2 attive per campo)
field_sessions (
  id uuid,
  field_id uuid references fields(id),
  player_1_id uuid references profiles(id),          -- utente registrato (obbligatorio)
  player_2_id uuid references profiles(id) nullable, -- secondo membro registrato (nullable)
  player_2_guest_name text nullable,                 -- nome ospite se player_2 non ha profilo
  started_at timestamptz,
  ended_at timestamptz nullable,                     -- null = sessione ancora attiva
  removed_by uuid references profiles(id) nullable   -- chi ha segnalato l'uscita dal campo
)

-- Lista d'attesa per campo
waiting_list (
  id uuid,
  field_id uuid references fields(id),
  player_1_id uuid references profiles(id),          -- utente registrato (obbligatorio)
  player_2_id uuid references profiles(id) nullable, -- secondo membro registrato (nullable)
  player_2_guest_name text nullable,                 -- nome ospite se player_2 non ha profilo
  position int,
  created_at timestamptz
)
```

### Constraint su ospiti
- `player_2_id` e `player_2_guest_name` sono mutualmente esclusivi: uno dei due deve essere non-null, oppure entrambi null (coppia da un solo membro? non consentito)
- Usare un CHECK constraint o validazione in Edge Function: `(player_2_id IS NOT NULL) != (player_2_guest_name IS NOT NULL)` oppure almeno uno dei due valorizzato

## RLS da implementare

- `presences`: tutti possono leggere, ogni utente modifica solo la propria
- `fields`: tutti possono leggere, solo staff/admin modificano manualmente lo stato
- `field_sessions`: tutti possono leggere; inserimento solo via Edge Function; **chiunque può chiudere una sessione attiva** (per la rimozione comunitaria)
- `waiting_list`: tutti possono leggere; ogni utente aggiunge solo se stesso come `player_1`; **chiunque può rimuovere qualsiasi voce** (rimozione comunitaria del campo)

## Edge Functions necessarie

- `join-field-queue` — aggiunge una coppia alla coda; se il campo ha posto (< 2 coppie), entra direttamente
- `leave-field-queue` — l'utente rimuove se stesso dalla coda
- `remove-couple-from-field` — chiunque segnala che una coppia ha lasciato il campo; chiude la `field_session`; promuove automaticamente la prima coppia in coda
- `auto-fill-field` — logica interna chiamata da `join-field-queue` e `remove-couple-from-field`; aggiorna `fields.status` e `couples_on_field`

## Componenti UI principali

1. **FieldMap** — griglia dei campi con stato live (`free` / `partial` / `occupied` / `maintenance`), mostra chi sta giocando per ogni campo
2. **CheckInButton** — pulsante toggle check-in/check-out nella home
3. **FieldDetail** — dettaglio singolo campo: chi sta giocando (con nomi ospiti), coda corrente, pulsante "Mettiti in coda", pulsante "Rimuovi dal campo" per le coppie attive
4. **JoinQueueModal** — selezione del secondo membro della coppia: cerca tra gli utenti registrati o inserisce nome ospite
5. **WaitingQueue** — lista d'attesa con posizione, nomi (inclusi ospiti), tempo di attesa
6. **AttendanceList** — lista di chi è in struttura oggi (nome + stato)

## Note implementative

- Usare `usePresence` hook per incapsulare la logica di check-in
- Usare `useFieldStatus` hook con subscription Realtime per lo stato dei campi e le sessioni attive
- La notifica "sei entrato in campo" può essere un toast in-app quando la coppia viene promossa dalla coda
- Il cambio di stato del campo deve essere **atomico** — `remove-couple-from-field` e `auto-fill-field` in una singola transazione
- I nomi degli ospiti (`player_2_guest_name`) sono solo display — non hanno `user_id` e non accumulano punti
- `removed_by` in `field_sessions` serve per storico/audit, non ha effetti sulla logica
- `couples_on_field` in `fields` è un campo derivato (denormalizzato) aggiornato atomicamente per evitare COUNT queries frequenti
