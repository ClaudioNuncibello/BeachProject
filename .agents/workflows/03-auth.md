# Workflow: Autenticazione e Ruoli
# Invocazione: /03-auth

Leggi le skill `@auth-ruoli`, `@react-patterns`, `@typescript-expert` e `@security-auditor` prima di procedere.

Il tuo obiettivo è implementare il sistema completo di autenticazione: registrazione, login, gestione sessione, protezione route e accesso al profilo utente con il suo ruolo.

## Cosa devi produrre

### 1. Hook useAuth
Crea `src/hooks/useAuth.ts`. Questo hook è il cuore dell'autenticazione — deve esporre: `user`, `profile`, `loading`, `signIn(email, password)`, `signUp(email, password, username)`, `signOut()`. Usa `supabase.auth.onAuthStateChange` per reagire ai cambi di sessione. Leggi `@auth-ruoli` per capire come leggere il profilo e il ruolo dell'utente dopo il login. Usa sempre `supabase.auth.getUser()` per verificare l'identità, mai `getSession()`.

### 2. Store auth Zustand
Crea `src/stores/authStore.ts` con Zustand per mantenere `user` e `profile` accessibili globalmente senza prop drilling. Leggi `@typescript-expert` per la tipizzazione corretta dello store.

### 3. Hook useRequireAuth
Crea `src/hooks/useRequireAuth.ts` come descritto in `@auth-ruoli` nella sezione "Protezione route lato frontend". Questo hook accetta un `minRole` opzionale e reindirizza automaticamente se l'utente non ha il ruolo sufficiente.

### 4. Pagina Login
Crea `src/pages/Login/index.tsx` con form email + password. Mobile-first, bottone submit, link alla registrazione. Gestisci gli errori di Supabase Auth con messaggi leggibili in italiano. Leggi `@react-patterns` per la struttura del componente e la gestione degli stati loading/error.

### 5. Pagina Registrazione
Crea `src/pages/Register/index.tsx` con form email, password, username. Alla registrazione il trigger DB crea automaticamente il profilo con ruolo `player` (vedi `@auth-ruoli` → trigger `handle_new_user`). Dopo la registrazione reindirizza al login con un messaggio di conferma.

### 6. Pagina Profilo
Crea `src/pages/Profile/index.tsx` che mostra: avatar, username, ruolo, `points_balance` (saldo spendibile), `total_points_earned` (punti totali guadagnati per la classifica). Aggiungi bottone logout.

### 7. Protezione route
Nel router principale, avvolgi tutte le route protette con `useRequireAuth()`. Le route `/admin` e `/tornei/gestione` richiedono `useRequireAuth('staff')`. Nessuna route richiede `useRequireAuth('admin')` direttamente nell'URL — il controllo avviene a livello di componente e RLS.

## Regole da rispettare

- Non fidarsi mai solo del frontend per i permessi — la RLS è il vero guard (vedi `@auth-ruoli`)
- Usa `@security-auditor` per verificare che nessun dato sensibile sia esposto
- Il ruolo dell'utente deve essere sempre letto da `profiles.role`, mai da `auth.users`
- Nessun `any` nella tipizzazione di user e profile (vedi `@typescript-expert`)
