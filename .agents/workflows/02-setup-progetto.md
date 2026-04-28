# Workflow: Setup Progetto
# Invocazione: /02-setup-progetto

Leggi le skill `@project-context`, `@react-patterns`, `@typescript-expert` e `@vercel-deployment` prima di procedere.

Il tuo obiettivo è inizializzare il progetto React + Vite + Supabase con la struttura di cartelle corretta, le dipendenze installate e i file di configurazione pronti.

## Cosa devi produrre

### 1. Inizializzazione progetto
Esegui i comandi per creare il progetto con Vite + React + TypeScript, installa Tailwind CSS, configura `tailwind.config.ts` e `index.css`. Installa le dipendenze core del progetto:
- `@supabase/supabase-js` — client Supabase
- `@tanstack/react-query` — server state management
- `zustand` — client state management
- `react-router-dom` — routing

### 2. Struttura cartelle
Crea la struttura di cartelle esatta descritta in `@project-context` sotto la sezione "Struttura cartelle progetto". Crea un file `index.ts` vuoto (barrel export) in ogni cartella.

### 3. Client Supabase
Crea il file `src/lib/supabase.ts` che inizializza il client Supabase usando le variabili d'ambiente `VITE_SUPABASE_URL` e `VITE_SUPABASE_ANON_KEY`. Crea il file `.env.local` con le variabili placeholder e aggiungi `.env.local` al `.gitignore`.

### 4. TypeScript types
Crea il file `src/types/database.ts` con tutti i tipi TypeScript che rispecchiano lo schema DB creato nel workflow `/01-schema-db`. Ogni tabella deve avere il suo tipo `Row`, `Insert` e `Update` seguendo il pattern di Supabase. Leggi `@typescript-expert` per i pattern di tipizzazione corretti.

### 5. Router e layout base
Crea il router principale in `src/main.tsx` con `react-router-dom`. Crea un componente `Layout` base con una bottom navigation mobile-first con 4 tab: Home (campi), Partite, FantaBeach, Profilo. Leggi `@react-patterns` per la struttura dei componenti.

### 6. Configurazione Vercel
Crea il file `vercel.json` con la configurazione per il deploy. Leggi `@vercel-deployment` per la configurazione corretta con Vite e le variabili d'ambiente.

### 7. PWA
Configura il progetto come PWA installabile usando `vite-plugin-pwa`. Crea il `manifest.json` con nome "Beach Volley Catania", icone e colori tema (sabbia/mare).

## Regole da rispettare

- TypeScript strict ovunque — nessun `any`, nessun `!` non necessario (vedi `@typescript-expert`)
- Nessuna chiave Supabase hardcoded — solo variabili d'ambiente
- Il layout deve essere mobile-first con Tailwind
- Ogni cartella deve avere il suo barrel export `index.ts`
