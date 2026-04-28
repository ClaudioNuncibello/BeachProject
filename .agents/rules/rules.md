# Beach Volley Catania — Project Rules
activation: always-on

## Stack — non deviare mai da queste tecnologie

- **Frontend**: React + Vite, mobile-first, PWA installabile
- **Backend/DB**: Supabase (Auth, Postgres, Realtime, Storage, RLS)
- **Hosting**: Vercel
- **Linguaggio**: TypeScript ovunque, niente JavaScript puro
- **Styling**: Tailwind CSS
- **State management**: Zustand per stato globale, React Query per server state

## Skill esterne consigliate (da @antigravity-awesome-skills)

Quando lavori su queste aree, usa le seguenti skill del repo sickn33/antigravity-awesome-skills:
- Design di sistema → `@architecture`
- Componenti React → `@react-patterns`
- TypeScript → `@typescript-expert`
- API Supabase → `@api-design-principles`
- Sicurezza/RLS → `@security-auditor`
- Deploy → `@vercel-deployment`
- Debug → `@debugging-strategies`

## Convenzioni codice

- Componenti React in PascalCase, file in kebab-case
- Ogni componente in cartella propria con `index.tsx`
- Hooks custom sempre con prefisso `use`
- Funzioni Supabase Edge in `/supabase/functions/`
- Variabili d'ambiente sempre in `.env.local`, mai hardcoded
- Commenti in **italiano** quando spiegano logica di business, in **inglese** per logica tecnica pura

## Struttura cartelle progetto

```
src/
├── components/        # UI components riusabili
├── features/          # Feature modules (campi, partite, fanta-beach)
│   ├── campi/
│   ├── partite/
│   └── fanta-beach/
├── hooks/             # Custom hooks
├── lib/               # Supabase client, utils
├── stores/            # Zustand stores
├── types/             # TypeScript types e interfaces
└── pages/             # Route pages
supabase/
├── functions/         # Edge functions
└── migrations/        # DB migrations con timestamp prefix
```

## Ruoli utente — rispettare sempre questa gerarchia

| Ruolo  | Valore DB | Permessi chiave                                        |
|--------|-----------|--------------------------------------------------------|
| player | `player`  | check-in, lista attesa, partite amatoriali, FantaBeach |
| staff  | `staff`   | tutto player + gestione tornei ufficiali               |
| admin  | `admin`   | tutto staff + gestione utenti, moderazione             |

- I controlli di ruolo vanno fatti **sia lato RLS Supabase sia lato UI**
- Non fidarsi mai solo del frontend per i permessi
- Usare sempre `auth.uid()` nelle RLS policy
- Il ruolo è salvato in `profiles.role` e letto con una funzione helper `getUserRole()`

## Supabase — regole fondamentali

- Usare sempre Row Level Security (RLS) su **ogni** tabella, nessuna eccezione
- Migrations in `/supabase/migrations/` con formato `YYYYMMDDHHMMSS_descrizione.sql`
- Usare Realtime **solo** per: stato campi live, bracket tornei, lista d'attesa
- Storage bucket `carte-giocatori` per immagini carte FantaBeach (pubblico in lettura)
- Usare sempre `supabase.auth.getUser()` lato server, mai `getSession()` per validare l'identità

## Sistema punti — valuta unica

I punti sono **una sola valuta** con doppia funzione: acquisto carte + classifica FantaBeach.

| Evento                        | Punti |
|-------------------------------|-------|
| Vittoria partita amatoriale   | 10    |
| Partecipazione torneo         | 20    |
| 3° posto torneo               | 50    |
| 2° posto torneo               | 70    |
| 1° posto torneo               | 100   |

**IMPORTANTE**: La classifica FantaBeach è basata sui punti totali **guadagnati** (`total_points_earned`), NON sul saldo corrente (`points_balance`). Comprare carte scala `points_balance` ma NON `total_points_earned`.
