# Workflow: Lista d'Attesa e Rotazione Turni
# Invocazione: /05-lista-attesa

Leggi le skill `@campi-turni`, `@auth-ruoli`, `@react-patterns`, `@api-design-principles` e `@typescript-expert` prima di procedere.

Il tuo obiettivo è implementare il sistema di lista d'attesa per i campi e la logica di rotazione turni: quando una coppia esce, la prima in coda entra automaticamente (auto-fill immediato, nessuna conferma richiesta).

> ⚠️ **La logica di rotazione NON è collegata alle partite.** Campi e partite sono sistemi indipendenti. Una coppia può uscire dal campo senza che esista una partita registrata nell'app. Il trigger dell'uscita è sempre `remove-couple-from-field`, mai la fine di una partita.

## Cosa devi produrre

### 1. Hook useWaitingList
Crea `src/hooks/useWaitingList.ts`. Gestisce la coda per un campo specifico. Espone: `queue` (lista ordinata per `created_at` FIFO), `myPosition` (posizione dell'utente nella coda, null se non in coda), `joinQueue(fieldId, partner)`, `leaveQueue()`. La coda è in Realtime — ogni aggiornamento si propaga immediatamente. Leggi `@campi-turni` per lo schema della tabella `waiting_list` e il formato del partner (utente registrato o nome ospite).

### 2. Edge Function join-field-queue
Crea `/supabase/functions/join-field-queue/index.ts`. Riceve: `field_id`, `player_2_id` (nullable) o `player_2_guest_name` (nullable) — uno dei due deve essere presente. Valida che `player_2_id` e `player_2_guest_name` siano mutualmente esclusivi. Controlla che `player_1` non sia già in coda per un altro campo. Se il campo ha meno di 2 coppie attive (`couples_on_field < 2`) → chiama `auto-fill-field` direttamente (la coppia entra subito). Altrimenti → crea il record in `waiting_list`. Aggiorna `presences.status` dell'utente a `waiting` se finisce in coda, o `on_field` se entra subito. Leggi `@campi-turni` per la logica completa.

### 3. Edge Function remove-couple-from-field
Crea `/supabase/functions/remove-couple-from-field/index.ts`. Può essere invocata da **chiunque** (modello comunitario — non solo i diretti interessati). Riceve: `session_id` (la `field_session` da chiudere). In una singola transazione atomica:
1. Setta `field_sessions.ended_at = now()` e registra `removed_by`
2. Aggiorna `presences.status` dei giocatori a `available`
3. Decrementa `fields.couples_on_field`
4. Chiama la logica `auto-fill-field`
Tutto in una transazione — se un passaggio fallisce, nessuno dei precedenti deve essere applicato. Leggi `@campi-turni` per la sequenza esatta e il vincolo che l'operazione deve essere atomica.

### 4. Edge Function auto-fill-field (logica interna)
Crea `/supabase/functions/auto-fill-field/index.ts` come funzione di supporto chiamata da `join-field-queue` e `remove-couple-from-field`. Logica:
1. Controlla se `fields.couples_on_field < 2`
2. Se sì e c'è almeno una coppia in `waiting_list` → prende la prima (FIFO per `created_at`)
3. Crea una nuova `field_session` per quella coppia
4. Rimuove la coppia dalla `waiting_list`
5. Aggiorna `presences.status` a `on_field` per i giocatori promossi
6. Aggiorna `fields.couples_on_field` e `fields.status` (`partial` o `occupied`)
7. Invia un toast in-app al `player_1` della coppia promossa: "Sei entrato in campo! 🏐"
L'auto-fill è immediato — non c'è un timer di accettazione. Leggi `@campi-turni` per i dettagli.

### 5. Componente JoinQueueModal
Crea `src/features/campi/JoinQueueModal/index.tsx`. Modal per accodarsi a un campo. Step 1: cerca il partner tra gli utenti registrati (con presenza attiva) oppure inserisce un nome ospite come testo libero. Step 2: anteprima coppia + conferma. Chiama la Edge Function `join-field-queue`. Se il campo ha posto, mostra il feedback "Sei entrato direttamente in campo!"; altrimenti mostra la posizione in coda. Leggi `@campi-turni` per la logica ospite e `@react-patterns` per modal multi-step.

### 6. Componente FieldDetail
Crea `src/features/campi/FieldDetail/index.tsx`. Dettaglio di un singolo campo. Mostra:
- Stato del campo e coppie attualmente in gioco (con nomi inclusi ospiti)
- Per ogni sessione attiva: bottone "Rimuovi dal campo" (visibile a tutti, non solo ai diretti interessati)
- Componente `WaitingQueue` con la coda corrente
- Bottone "Mettiti in coda" se il campo è `occupied` e l'utente non è già in coda
Leggi `@campi-turni` per il modello comunitario di rimozione.

### 7. Componente WaitingQueue
Crea `src/features/campi/WaitingQueue/index.tsx`. Mostra la coda per un campo specifico. Lista verticale con: posizione numerata, avatar/nome di ogni coppia in attesa (mostrando anche nomi ospiti), tempo di attesa stimato. La posizione dell'utente loggato è evidenziata. Bottone "Esci dalla coda" se l'utente è già in coda. Leggi `@react-patterns` per liste con stato evidenziato.

### 8. Integrazione in FieldMap
Aggiorna il componente `FieldMap` creato nel workflow `/04-campi-checkin`. Ogni card campo ora è cliccabile e apre il `FieldDetail`. Se il campo è `occupied`, mostra anche il numero di coppie in attesa.

## Regole da rispettare

- La logica di rotazione (chiusura sessione + auto-fill) deve essere **atomica** — usa sempre le Edge Functions, mai due chiamate separate dalla UI (vedi `@campi-turni`)
- L'auto-fill è **immediato e automatico** — non c'è un timer o una schermata di accettazione
- La coda è FIFO rigoroso — ordinata per `created_at`, nessuna eccezione
- Un utente non può essere in coda per più campi contemporaneamente — controlla nella Edge Function `join-field-queue`
- `remove-couple-from-field` può essere invocata da chiunque — non filtrare per utente (modello comunitario)
- **I campi sono indipendenti dalle partite** — `remove-couple-from-field` non tocca mai la tabella `matches`
