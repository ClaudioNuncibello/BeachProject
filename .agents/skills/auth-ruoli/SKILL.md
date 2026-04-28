---
name: auth-ruoli
description: Skill per la gestione dell'autenticazione, dei ruoli utente e delle RLS policy Supabase. Usa quando implementi login/registrazione, controlli di permesso, middleware di protezione route, o scrivi RLS policy per nuove tabelle.
---

# Skill: Auth & Ruoli

## Sistema di autenticazione

- Usare **Supabase Auth** con email + password (v1)
- La registrazione crea automaticamente un record in `profiles` via trigger DB
- Il profilo contiene: `id`, `username`, `avatar_url`, `role`, `points_balance`, `total_points_earned`
- Il ruolo di default per ogni nuovo utente è `player`

### Trigger di creazione profilo

```sql
create or replace function handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, username, role, points_balance, total_points_earned)
  values (
    new.id,
    new.raw_user_meta_data->>'username',
    'player',
    0,
    0
  );
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure handle_new_user();
```

## Gerarchia ruoli

```
admin
  └── staff
        └── player
```

- Ogni ruolo superiore ha tutti i permessi del ruolo inferiore
- Non usare numeri per i ruoli — usare sempre le stringhe `'player'`, `'staff'`, `'admin'`

## Helper funzione ruolo (usare sempre questa)

```sql
-- Funzione helper da usare nelle RLS policy
create or replace function get_user_role()
returns text as $$
  select role from public.profiles where id = auth.uid();
$$ language sql security definer stable;

-- Funzione per check ruolo minimo
create or replace function is_staff_or_above()
returns boolean as $$
  select get_user_role() in ('staff', 'admin');
$$ language sql security definer stable;

create or replace function is_admin()
returns boolean as $$
  select get_user_role() = 'admin';
$$ language sql security definer stable;
```

## Pattern RLS standard

### Tabella solo lettura per tutti, scrittura solo admin
```sql
alter table cards enable row level security;

create policy "cards_select_all" on cards
  for select using (true);

create policy "cards_insert_admin" on cards
  for insert with check (is_admin());

create policy "cards_update_admin" on cards
  for update using (is_admin());
```

### Tabella con dati personali
```sql
alter table user_cards enable row level security;

create policy "user_cards_select_own" on user_cards
  for select using (auth.uid() = user_id);

-- L'insert avviene solo via Edge Function con service_role key, non dalla UI
```

### Tabella con lettura pubblica e scrittura owner
```sql
alter table presences enable row level security;

create policy "presences_select_all" on presences
  for select using (true);  -- tutti vedono chi è presente

create policy "presences_insert_own" on presences
  for insert with check (auth.uid() = user_id);

create policy "presences_update_own" on presences
  for update using (auth.uid() = user_id);
```

## Protezione route lato frontend

```typescript
// hooks/useRequireAuth.ts
export function useRequireAuth(minRole?: 'player' | 'staff' | 'admin') {
  const { user, profile, loading } = useAuth()
  const navigate = useNavigate()

  useEffect(() => {
    if (!loading && !user) {
      navigate('/login')
      return
    }
    if (minRole && profile && !hasRole(profile.role, minRole)) {
      navigate('/')  // redirect a home se ruolo insufficiente
    }
  }, [user, profile, loading, minRole])

  return { user, profile, loading }
}

// Uso:
// Nelle pagine admin: useRequireAuth('admin')
// Nelle pagine staff: useRequireAuth('staff')
// In tutte le pagine protette: useRequireAuth()
```

## Cosa NON fare

- ❌ MAI usare `getSession()` lato server per verificare l'identità — è falsificabile
- ❌ MAI fare controlli di ruolo solo sul frontend senza RLS
- ❌ MAI esporre la `service_role` key nel frontend
- ❌ MAI scrivere su `point_transactions`, `user_cards`, `profiles.role` direttamente dalla UI
- ❌ MAI skippare RLS su una tabella "perché tanto è solo lettura"

## Cambio ruolo utente

Il cambio ruolo (es. promuovere qualcuno a staff) avviene solo:
- Tramite pannello admin nell'app (solo admin possono farlo)
- L'update di `profiles.role` è protetto da RLS: solo admin possono modificare il campo role

```sql
create policy "profiles_update_role_admin_only" on profiles
  for update using (
    -- l'utente può aggiornare il proprio profilo
    auth.uid() = id
    -- ma non può modificare il campo role (a meno che non sia admin)
    -- questo si gestisce con una policy separata o una funzione
  );
```
