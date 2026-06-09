# ConsoMate

## Pré-requis Supabase

Exécutez ce SQL dans l'éditeur SQL de votre projet Supabase :

```sql
create table public.readings (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  type text not null check (type in ('water', 'electricity')),
  value numeric not null,
  unit text not null,
  reading_date date not null default current_date,
  note text,
  created_at timestamptz not null default now()
);

alter table public.readings enable row level security;

create policy "Users can manage their own readings"
on public.readings
for all
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

## Utilisation

1. Ouvrez la page dans votre navigateur.
2. Collez l'URL du projet Supabase et la clé anonyme dans la section de configuration.
3. Créez un compte ou connectez-vous.
4. Ajoutez ensuite vos relevés d'eau ou d'électricité.
