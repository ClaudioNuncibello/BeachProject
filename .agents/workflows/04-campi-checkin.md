# Workflow: Campi e Check-in
# Invocazione: /04-campi-checkin

Leggi le skill `@campi-turni`, `@auth-ruoli`, `@react-patterns`, `@typescript-expert` e `@api-design-principles` prima di procedere.

Il tuo obiettivo è implementare la home operativa dell'app: il check-in degli utenti, la visualizzazione in tempo reale dello stato dei campi e la lista di chi è presente in struttura oggi.

## Cosa devi produrre

### 1. Hook usePresence
Crea `src/hooks/usePresence.ts`. Gestisce tutto il ciclo di vita della presenza dell'utente: `checkIn()`, `checkOut()`, `isCheckedIn`, `myPresence`. Il check-in crea un record in `presences` con `status: 'available'`. Il check-out setta `status: 'checked_out'`. Un utente può avere un solo check-in attivo alla volta — aggiungi questo controllo prima di inserire. Leggi `@campi-turni` per lo schema esatto della tabella `presences`.

### 2. Hook useFieldStatus
Crea `src/hooks/useFieldStatus.ts`. Si iscrive in tempo reale allo stato di tutti i campi tramite Supabase Realtime (`supabase.channel().on('postgres_changes',...)`). Carica `fields` e le `field_sessions` attive (`ended_at IS NULL`) con JOIN ai profili. Espone: `fields`, `activeSessions` (chi sta giocando per campo), `loading`. Lo stato di ogni campo (`free` / `partial` / `occupied` / `maintenance`) deve aggiornarsi automaticamente senza refresh. Per sapere dove si trova un utente usa sempre JOIN tra `presences` e `field_sessions` — non aggiungere `field_id` in `presences`. Leggi `@campi-turni` per la query corretta con JOIN.

### 3. Hook useAttendance
Crea `src/hooks/useAttendance.ts`. Carica la lista di tutti gli utenti con check-in attivo oggi (`presences` con `status != 'checked_out'` e `checked_in_at` di oggi). Aggiorna in Realtime quando qualcuno entra o esce. Espone: `attendees` (con profilo + status), `count`. Leggi `@campi-turni` per la query corretta.

### 4. Componente CheckInButton
Crea `src/features/campi/CheckInButton/index.tsx`. Un bottone toggle prominente nella home: se l'utente non è in struttura mostra "Sono arrivato 🏐", se è già in struttura mostra "Vado via". Gestisce loading state durante la chiamata. Leggi `@react-patterns` per i pattern di bottoni con stato.

### 5. Componente FieldMap
Crea `src/features/campi/FieldMap/index.tsx`. Mostra una griglia di card, una per ogni campo. Ogni card mostra:
- Nome campo
- Stato con colore: verde = `free`, giallo = `partial`, rosso = `occupied`, grigio = `maintenance`
- Se `partial` o `occupied`: nomi delle coppie attive sul campo (inclusi ospiti `player_2_guest_name`)
- Numero di persone in attesa (da `waiting_list`)
- Se `occupied`: bottone "Mettiti in coda" che porta al `FieldDetail`
I dati arrivano da `useFieldStatus` e si aggiornano in Realtime. Leggi `@react-patterns` per la struttura delle card e `@campi-turni` per i 4 stati del campo.

### 6. Componente AttendanceList
Crea `src/features/campi/AttendanceList/index.tsx`. Lista scorrevole di chi è in struttura adesso con avatar, username e stato (`available` / `on_field` / `waiting`). Mostra il contatore totale in cima. Si aggiorna in Realtime.

### 7. Pagina Home
Crea `src/pages/Home/index.tsx`. Compone i tre componenti sopra in questo ordine: `CheckInButton` in cima, `FieldMap` al centro, `AttendanceList` in fondo. Proteggi la pagina con `useRequireAuth()`.

## Regole da rispettare

- Lo stato dei campi DEVE essere Realtime — non usare polling (vedi `@campi-turni`)
- Il check-in deve essere idempotente: se l'utente prova a fare check-in due volte, non creare un secondo record
- I 4 stati del campo sono: `free`, `partial`, `occupied`, `maintenance` — non usare `playing` (vedi `@campi-turni`)
- `presences` non ha `field_id` — usa sempre JOIN con `field_sessions` per sapere dove si trova un utente
- Usa `@api-design-principles` per strutturare bene le query con join (presenze + profili + field_sessions)
- Mobile-first: la FieldMap deve funzionare bene su schermi piccoli (cards verticali)
