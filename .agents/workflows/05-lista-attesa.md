# Workflow: Lista d'Attesa e Rotazione Turni
# Invocazione: /05-lista-attesa

Leggi le skill `@campi-turni`, `@auth-ruoli`, `@react-patterns`, `@api-design-principles` e `@typescript-expert` prima di procedere.

Il tuo obiettivo è implementare il sistema di lista d'attesa per i campi occupati e la logica di rotazione turni: vincitore resta, perdente lascia e il primo in coda entra.

## Cosa devi produrre

### 1. Hook useWaitingList
Crea `src/hooks/useWaitingList.ts`. Gestisce la coda per un campo specifico. Espone: `queue` (lista ordinata per `created_at`), `myPosition` (posizione dell'utente nella coda, null se non in coda), `joinQueue(fieldId)`, `leaveQueue()`. La coda è in Realtime — ogni aggiornamento si propaga immediatamente. Leggi `@campi-turni` per lo schema della tabella `waiting_list`.

### 2. Edge Function handle-match-end
Crea `/supabase/functions/handle-match-end/index.ts`. Questa è la funzione critica che gestisce la fine di una partita in modo atomico. Quando viene invocata deve: (1) segnare la partita come completata, (2) rimettere i giocatori a `status: 'available'`, (3) rimettere il campo a `status: 'free'`, (4) notificare in-app il primo utente in lista d'attesa, (5) rimuoverlo dalla coda e assegnargli il campo. Tutto in una transazione — se un passaggio fallisce, nessuno dei precedenti deve essere applicato. Leggi `@campi-turni` per la logica esatta e `@auth-ruoli` per verificare che solo i partecipanti della partita possano invocarla.

### 3. Componente WaitingQueue
Crea `src/features/campi/WaitingQueue/index.tsx`. Mostra la coda per un campo specifico. Lista verticale con: posizione numerata, avatar e username di ogni utente in attesa. La posizione dell'utente loggato è evidenziata. Bottone "Mettimi in coda" se il campo è occupato e l'utente non è già in coda. Bottone "Esci dalla coda" se l'utente è già in coda. Leggi `@react-patterns` per liste con stato evidenziato.

### 4. Notifica "Sei il prossimo"
Implementa una notifica in-app (toast) che appare quando è il turno dell'utente. Usa Supabase Realtime per ascoltare i cambiamenti sulla propria riga in `waiting_list`. Quando `notified_at` viene valorizzato, mostra il toast: "È il tuo turno! Vai al [nome campo] 🏐". Il toast deve restare visibile finché l'utente non lo chiude. Leggi `@api-design-principles` per le subscription su righe specifiche.

### 5. Integrazione in FieldMap
Aggiorna il componente `FieldMap` creato nel workflow `/04-campi-checkin`. Ogni card campo ora deve mostrare anche: numero di persone in attesa, e se il campo è occupato un bottone "Mettiti in coda" che apre il componente `WaitingQueue`. Leggi `@react-patterns` per l'integrazione di componenti esistenti.

## Regole da rispettare

- La logica di rotazione (aggiornamento campo + lista attesa) deve essere **atomica** — usa sempre la Edge Function, mai due chiamate separate dalla UI (vedi `@campi-turni`)
- La coda deve essere FIFO rigoroso — ordinata per `created_at`, nessuna eccezione
- Un utente non può essere in coda per più campi contemporaneamente — aggiungi questo controllo nella Edge Function
- Il timeout "sei il prossimo" (3 minuti per accettare) va gestito lato Edge Function, non lato client
