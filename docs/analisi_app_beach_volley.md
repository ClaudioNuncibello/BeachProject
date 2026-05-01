# 🏐 Beach Volley Catania — Analisi Completa dell'App

> **Stato progetto**: Fase di progettazione — nessun codice scritto ancora.
> **V1 in scope**: La Struttura + Partite Amatoriali
> **V1 out of scope**: Tornei, FantaBeach (placeholder "Coming Soon")

---

## 1. Visione Generale

Una **web app mobile-first** (installabile come PWA) per la community di beach volley di una struttura a Catania. L'obiettivo è digitalizzare la gestione informale della struttura: chi è presente, quale campo è libero, la coda d'attesa, e il tracciamento delle partite amatoriali con un sistema punti.

### Stack Tecnico

| Layer | Tecnologia |
|---|---|
| Frontend | React + Vite + TypeScript + Tailwind CSS |
| Backend | Supabase (Auth, DB PostgreSQL, Realtime, Storage) |
| Deploy | Vercel |
| Tipo app | PWA mobile-first |

---

## 2. Utenti e Ruoli

Solo **giocatori abituali della struttura**. Nessun accesso pubblico esterno nella v1.

```
admin
  └── staff
        └── player
```

| Ruolo | Permessi |
|---|---|
| `player` | Check-in/out, accodarsi ai campi, registrare/confermare partite, vedere classifica |
| `staff` | Tutto di `player` + gestire tornei, risolvere dispute tra giocatori |
| `admin` | Tutto di `staff` + creare eventi FantaBeach, assegnare punti, cambiare ruoli |

**Registrazione**: email + password via Supabase Auth. Un trigger DB crea automaticamente il profilo `player` alla registrazione.

---

## 3. Le 4 Macro-Sezioni

```
┌─────────────────────────────────────────────────────────┐
│  Nav Bar (bottom mobile)                                │
│  [La Struttura] [Partite] [Tornei 🔜] [FantaBeach 🔜]   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Sezione 1 — La Struttura (Home Operativa)

La sezione centrale dell'app nella v1. Gestisce presenze, campi e code in tempo reale.

### 4.1 Concetti chiave

> ⚠️ **Campi e Partite sono SEPARATI.** Occupare un campo NON crea una partita. La partita si registra separatamente, a posteriori, quando i giocatori lo decidono.

- Un campo ospita **max 2 coppie** contemporaneamente (4 giocatori)
- La coda avanza **automaticamente** (FIFO): quando una coppia esce, la prima in coda entra subito
- **Chiunque** può segnalare l'uscita di una coppia dal campo (modello comunitario)
- Il secondo membro di una coppia può essere un **ospite** (solo nome testuale, senza profilo)

### 4.2 Stati di un campo

```
free (0 coppie) → partial (1 coppia) → occupied (2 coppie)
                                              ↓
                                       maintenance (fuori uso)
```

### 4.3 Stati di una presenza utente

```
[Arrivo in struttura]
        ↓
   checked_in (available)
        ↓
  si accoda → waiting
        ↓
  entra in campo → on_field
        ↓
  esce dal campo → available  (può riaccodarsi)
        ↓
   check-out → checked_out
```

### 4.4 Flusso utente — Check-in e Coda

```
1. L'utente apre l'app e vede la Home (La Struttura)
2. Preme [Check-in] → status diventa 'available', appare nella lista presenze
3. Vede la mappa campi live:
   - Campo 1: occupied (Marco & Luca | Sara & Gianni) — coda: 2 coppie
   - Campo 2: partial (Andrea & Ospite "Fabio") — 1 posto libero
   - Campo 3: free
4. L'utente tocca [Campo 2] → apre il FieldDetail
5. Vede chi sta giocando e la coda corrente
6. Preme [Mettiti in coda]
   → apre JoinQueueModal
   → seleziona il partner: cerca tra utenti registrati O inserisce nome ospite
   → conferma la coppia
7. Se il campo ha posto (<2 coppie) → entra subito (nuova field_session, status → on_field)
   Se il campo è occupied → finisce in waiting_list (status → waiting)
8. Quando una coppia esce (segnalata da chiunque), la prima in coda viene promossa:
   → toast in-app: "Sei entrato in campo!"
   → status → on_field
9. Al termine, l'utente (o chiunque) preme [Rimuovi dal campo]
   → field_session.ended_at = now()
   → promozione automatica prossima coppia in coda
10. L'utente preme [Check-out] → status → checked_out
```

### 4.5 Componenti UI principali

| Componente | Descrizione |
|---|---|
| `CheckInButton` | Toggle check-in / check-out nella home |
| `FieldMap` | Griglia campi con stato live (free/partial/occupied/maintenance) |
| `FieldDetail` | Dettaglio campo: sessioni attive, coda, azioni |
| `JoinQueueModal` | Selezione partner (utente o ospite) prima di accodarsi |
| `WaitingQueue` | Lista d'attesa con posizione e tempo di attesa |
| `AttendanceList` | Chi è in struttura oggi e il loro status |

### 4.6 Edge Functions (Backend)

| Funzione | Cosa fa |
|---|---|
| `join-field-queue` | Aggiunge coppia alla coda; se c'è posto, entra subito |
| `leave-field-queue` | L'utente si rimuove dalla coda |
| `remove-couple-from-field` | Chiunque segnala uscita; chiude field_session; promuove coda |
| `auto-fill-field` | Logica interna: aggiorna status campo e couples_on_field atomicamente |

---

## 5. Sezione 2 — Partite Amatoriali

### 5.1 Concetti chiave

> ⚠️ **Le partite si registrano A POSTERIORI.** Si gioca fisicamente, poi uno dei 4 giocatori apre l'app e inserisce il risultato. Non c'è tracking live.

- Doppia validazione: entrambi i capitani devono confermare il risultato
- Sistema ranked/unranked automatico (max 1 ranked al giorno per utente)
- Solo le partite ranked assegnano punti (+10 ai vincitori)

### 5.2 Flusso utente — Registrazione Partita

```
1. Dopo aver giocato fisicamente, uno dei 4 apre l'app → sezione "Partite"
2. Preme [+ Registra Partita] → apre RegisterMatchModal
3. Step 1 — Selezione giocatori:
   - Team A: [Io (autom.)] + [cerca secondo giocatore]
   - Team B: [cerca giocatore 3] + [cerca giocatore 4]
   - Campo (opzionale, solo informativo): seleziona dal menu
4. Step 2 — Inserimento punteggio set per set:
   - Set 1: Team A [21] — Team B [18]
   - Set 2: Team A [19] — Team B [21]
   - Set 3: Team A [15] — Team B [12]
   - Il sistema calcola automaticamente il vincitore (primo a 2 set)
5. Step 3 — Anteprima e conferma:
   - Badge prominente: "RANKED ✅" oppure "UNRANKED ⚡"
     (determinato automaticamente: se uno dei 4 ha già una ranked oggi → unranked)
   - Riepilogo risultato e giocatori
   - [Conferma e Invia]
6. La partita viene creata in stato 'pending_validation'
7. Il capitano avversario (primo giocatore del Team B) riceve notifica in-app
```

### 5.3 Flusso utente — Validazione Risultato

```
Il capitano avversario apre l'app e vede:

┌─────────────────────────────────────────────┐
│  ⚠️ Conferma risultato partita              │
│  Marco & Luca vs Sara & Gianni              │
│  Risultato: 2-1 (21-18, 19-21, 15-12)      │
│  Tipo: RANKED                               │
│                                             │
│  [✅ Confermo]    [❌ Contesto]              │
└─────────────────────────────────────────────┘

CASO A — Confermo:
  → partita status: 'completed'
  → se ranked: vincitori +10 punti (points_balance + total_points_earned)
  → point_transaction creata via Edge Function

CASO B — Contesto:
  → partita status: 'disputed'
  → notifica a staff/admin per risoluzione manuale
  → lo staff vede la partita in DisputedMatchAdmin
  → lo staff risolve manualmente (sceglie il risultato corretto)

CASO C — Nessuna risposta entro 24 ore:
  → cron job (pg_cron) marca la partita come 'completed' automaticamente
  → punti assegnati come da risultato inserito
```

### 5.4 Regola Ranked/Unranked

```
Al momento della creazione della partita:

Per ogni giocatore in [team_a + team_b]:
  Se esiste una partita ranked con status IN
  ('pending_validation', 'disputed', 'completed')
  registrata OGGI (fuso Europe/Rome)
  → partita = UNRANKED

Se tutti e 4 non hanno ancora una ranked oggi
  → partita = RANKED
```

> Nota: si controlla anche le partite in `pending_validation` e `disputed` per evitare abusi (registrare più partite ranked prima che vengano validate).

### 5.5 Storico Partite

```
L'utente nella sezione Partite vede:

[Filtra: Tutte | Ranked | Unranked | In attesa]

┌───────────────────────────────────────┐
│ 🟢 RANKED · Oggi 15:30               │
│ Marco & Luca  2-1  Sara & Gianni     │
│ Stato: ✅ Completata · +10 pt        │
├───────────────────────────────────────┤
│ ⚡ UNRANKED · Ieri 17:00             │
│ Marco & Luca  0-2  Andrea & Paolo    │
│ Stato: ✅ Completata · +0 pt         │
├───────────────────────────────────────┤
│ ⏳ RANKED · Oggi 16:00               │
│ Marco & Gianni  2-0  ...             │
│ Stato: In attesa di conferma         │
└───────────────────────────────────────┘
```

### 5.6 Componenti UI principali

| Componente | Descrizione |
|---|---|
| `RegisterMatchModal` | Form multi-step: giocatori → punteggio → anteprima ranked |
| `PendingValidationBanner` | Banner per il capitano avversario con Conferma/Contesta |
| `MatchHistory` | Storico partite filtrabili con risultati e punti |
| `DisputedMatchAdmin` | Vista staff per risolvere dispute |

### 5.7 Edge Functions (Backend)

| Funzione | Cosa fa |
|---|---|
| `register-match` | Registra partita, determina ranked, notifica capitano avversario |
| `confirm-match-result` | Capitano avversario conferma → completed + punti |
| `dispute-match-result` | Capitano avversario contesta → disputed + notifica staff |
| `resolve-match-dispute` | Staff risolve manualmente |
| `assign-match-points` | Assegna punti solo se ranked=true |

---

## 6. Sezione 3 — Tornei Ufficiali 🔜 Coming Soon

> **Non implementata nella v1.** Mostrare placeholder nella navigazione.

### Visione futura
- Creazione torneo da parte dei moderatori (nome, data, formato bracket)
- Bracket/tabellone aggiornato in real-time dallo staff
- Piazzamento finale → punti assegnati ai partecipanti
- I punti tornei si sommano a quelli delle partite nella classifica globale

---

## 7. Sezione 4 — FantaBeach 🔜 Coming Soon

> **Non implementata nella v1.** Mostrare placeholder nella navigazione.

### Visione futura
Un gioco **virtuale** stile fantacalcio applicato al beach volley locale. Completamente separato dai tornei fisici.

**Ciclo del gioco:**
```
1. Utente guadagna punti (da partite ranked e tornei)
2. Compra carte dei giocatori della struttura nello Shop
3. Costruisce una rosa di 3 coppie (6 carte)
4. Quando un admin crea un Evento FantaBeach, schiera una coppia
5. L'admin assegna manualmente i punti dopo l'evento
```

**Componenti futuri:** CardGrid, ShopPage, RosterEditor, FantaEventList, LineupSelector, FantaEventAdminPanel, Leaderboard.

---

## 8. Sistema Punti — v1

| Fonte | Punti | Note |
|---|---|---|
| Partita ranked vinta | +10 | Max 1 ranked al giorno per utente |
| Partita ranked persa | +0 | Nessuna penalità |
| Partita unranked | +0 | Solo per divertimento |

### Due campi distinti nel profilo:
- **`total_points_earned`** → classifica globale (non scende mai, neanche comprando carte)
- **`points_balance`** → saldo futuro per lo shop carte FantaBeach (nella v1 non è usato attivamente)

### Classifica
```
#  Avatar  Nome          Punti Totali
1  👤      Marco R.      340 pt
2  👤      Sara L.       290 pt
3  👤      Andrea P.     210 pt
...
```

---

## 9. Schema Database Completo

### Tabelle principali (v1)

```sql
-- Profili utente
profiles (
  id uuid,                    -- = auth.users.id
  username text,
  avatar_url text nullable,
  role text,                  -- 'player' | 'staff' | 'admin'
  points_balance int,         -- saldo per shop (futuro)
  total_points_earned int     -- classifica globale
)

-- Presenze giornaliere
presences (
  id uuid,
  user_id uuid → profiles,
  status text,                -- 'available' | 'on_field' | 'waiting' | 'checked_out'
  checked_in_at timestamptz,
  checked_out_at timestamptz nullable
)

-- Campi della struttura
fields (
  id uuid,
  name text,                  -- 'Campo 1', 'Campo 2', ...
  status text,                -- 'free' | 'partial' | 'occupied' | 'maintenance'
  couples_on_field int        -- 0, 1 o 2
)

-- Sessioni attive sul campo (max 2 per campo)
field_sessions (
  id uuid,
  field_id uuid → fields,
  player_1_id uuid → profiles,
  player_2_id uuid nullable → profiles,
  player_2_guest_name text nullable,
  started_at timestamptz,
  ended_at timestamptz nullable,      -- null = ancora in corso
  removed_by uuid nullable → profiles
)

-- Lista d'attesa
waiting_list (
  id uuid,
  field_id uuid → fields,
  player_1_id uuid → profiles,
  player_2_id uuid nullable → profiles,
  player_2_guest_name text nullable,
  position int,
  created_at timestamptz
)

-- Partite amatoriali
matches (
  id uuid,
  field_id uuid nullable → fields,   -- solo informativo
  team_a_ids uuid[],
  team_b_ids uuid[],
  captain_a_id uuid → profiles,
  captain_b_id uuid → profiles,
  ranked bool,
  status text,    -- 'pending_validation' | 'disputed' | 'completed'
  winner_team text nullable,          -- 'a' | 'b'
  registered_by uuid → profiles,
  opponent_captain_confirmed bool,
  created_at timestamptz,
  completed_at timestamptz nullable
)

-- Set delle partite
match_sets (
  id uuid,
  match_id uuid → matches,
  set_number int,
  score_a int,
  score_b int,
  winner text nullable        -- 'a' | 'b'
)

-- Transazioni punti (universale: partite + tornei + FantaBeach)
point_transactions (
  id uuid,
  user_id uuid → profiles,
  amount int,
  reason text,                -- 'match_win' | 'tournament_1st' | 'fanta_event' ...
  reference_id uuid,
  created_at timestamptz
)
```

---

## 10. Sicurezza — RLS Policy

| Tabella | Lettura | Scrittura |
|---|---|---|
| `presences` | Tutti | Solo il proprio record |
| `fields` | Tutti | Solo staff/admin (manuale) |
| `field_sessions` | Tutti | Insert via Edge Function; **chiunque può chiudere** |
| `waiting_list` | Tutti | Insert solo come player_1; **chiunque può rimuovere** |
| `matches` | Tutti | Solo partecipanti possono aggiornare |
| `match_sets` | Tutti | Solo partecipanti della partita |
| `point_transactions` | Solo le proprie | Solo Edge Functions (service_role) |
| `profiles.role` | Tutti | Solo admin |

---

## 11. Autenticazione — Flusso Login/Registrazione

```
[Schermata di apertura app]
        ↓
  Non autenticato → Landing/Login page
        ↓
  ┌─────────────────┐    ┌──────────────────┐
  │    Accedi       │    │   Registrati     │
  │  email+password │    │  email+password  │
  │                 │    │  + username      │
  └────────┬────────┘    └────────┬─────────┘
           │                     │
           │              Supabase Auth crea user
           │              Trigger DB crea profilo
           │              (role: 'player', points: 0)
           ↓                     ↓
      Supabase Auth         JWT session
           ↓
   App carica profilo da 'profiles'
           ↓
   Redirect a Home (La Struttura)
```

---

## 12. Architettura Realtime

Supabase Realtime viene usato per mantenere la UI aggiornata senza refresh manuale:

| Dato | Subscription |
|---|---|
| Stato campi (`fields`) | Hook `useFieldStatus` — aggiornamento live |
| Sessioni attive (`field_sessions`) | Hook `useFieldStatus` — chi sta giocando |
| Lista d'attesa (`waiting_list`) | Hook `useFieldStatus` — posizione in coda |
| Presenze (`presences`) | Hook `usePresence` — chi è in struttura |
| Partite pendenti (`matches`) | Notifica in-app per validazione |

---

## 13. Navigazione dell'App

```
┌─────────────────────────────────────────┐
│  HEADER: Logo + [Profilo] [Notifiche]   │
├─────────────────────────────────────────┤
│                                         │
│         CONTENUTO PAGINA               │
│                                         │
├─────────────────────────────────────────┤
│  BOTTOM NAV (mobile):                   │
│  🏠 Struttura | 🏐 Partite | 🏆 Tornei* | 🃏 Fanta* │
└─────────────────────────────────────────┘

* = placeholder "Coming Soon" nella v1
```

### Pagine principali v1:
1. `/` — La Struttura (Home operativa)
2. `/partite` — Partite Amatoriali
3. `/tornei` — Placeholder Coming Soon
4. `/fanta` — Placeholder Coming Soon
5. `/profilo` — Profilo utente + punti + storico
6. `/login` — Accesso
7. `/register` — Registrazione

---

## 14. Roadmap

### V1 (In Scope)
- [x] Sistema Auth (registrazione/login)
- [ ] Profili utente
- [ ] La Struttura: check-in/out, mappa campi live
- [ ] La Struttura: coda e rotazione campi
- [ ] Partite: registrazione a posteriori
- [ ] Partite: sistema ranked/unranked
- [ ] Partite: doppia validazione e dispute
- [ ] Sistema punti (solo da partite ranked)
- [ ] Classifica globale

### Futuro (Out of Scope v1)
- [ ] Tornei ufficiali con bracket
- [ ] FantaBeach: carte collezionabili e shop
- [ ] FantaBeach: rosa e eventi
- [ ] Timer turni sul campo
- [ ] Notifiche push (PWA)
- [ ] Statistiche avanzate per giocatore

---

## 15. Principi Architetturali Chiave

1. **Separazione dei contesti**: Campi ≠ Partite. Sono sistemi indipendenti.
2. **Backend-first per dati sensibili**: `point_transactions`, `profiles.role` e acquisti carte si scrivono **solo via Edge Functions** con service_role key. Mai dalla UI.
3. **RLS ovunque**: Nessuna tabella senza Row Level Security, nemmeno quelle "solo lettura".
4. **Atomicità**: Le operazioni sui campi (rimozione + promozione coda) avvengono in una singola transazione per evitare inconsistenze.
5. **Fuso Europe/Rome**: Tutte le regole "giornaliere" (ranked, check-in) usano il timezone italiano.
6. **Modello comunitario**: La rimozione dal campo è aperta a tutti gli utenti — nessun permesso speciale. Funziona in un contesto di giocatori abituali.
