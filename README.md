# ConsoMate

## Pré-requis Supabase

Exécutez ce SQL dans l'éditeur SQL de votre projet Supabase :

Si Supabase affiche `relation "readings" already exists`, vous avez lancé une ancienne version du SQL. Utilisez le bloc ci-dessous tel quel : il contient `create table if not exists` pour conserver la table existante.

```sql
create table if not exists public.readings (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  type text not null check (type in ('water', 'electricity')),
  value numeric not null,
  unit text not null,
  reading_date timestamptz not null default now(),
  reading_datetime_local timestamp not null default (now() at time zone 'Europe/Paris'),
  note text,
  created_at timestamptz not null default now()
);

create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text not null unique,
  created_at timestamptz not null default now()
);

create table if not exists public.mates (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid not null references auth.users(id) on delete cascade,
  mate_id uuid not null references auth.users(id) on delete cascade,
  created_at timestamptz not null default now(),
  unique (owner_id, mate_id),
  check (owner_id <> mate_id)
);

alter table public.readings enable row level security;
alter table public.profiles enable row level security;
alter table public.mates enable row level security;

alter table public.readings
alter column reading_date drop default;

alter table public.readings
alter column reading_date type timestamptz
using reading_date::timestamptz;

alter table public.readings
alter column reading_date set default now();

alter table public.readings
add column if not exists reading_datetime_local timestamp;

update public.readings
set reading_datetime_local = reading_date at time zone 'Europe/Paris'
where reading_datetime_local is null;

alter table public.readings
alter column reading_datetime_local set default (now() at time zone 'Europe/Paris');

alter table public.readings
alter column reading_datetime_local set not null;

create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.profiles (id, email)
  values (new.id, lower(new.email))
  on conflict (id) do update
  set email = excluded.email;

  return new;
end;
$$;

drop trigger if exists on_auth_user_created on auth.users;

create trigger on_auth_user_created
after insert on auth.users
for each row execute function public.handle_new_user();

drop policy if exists "Users can manage their own readings" on public.readings;
drop policy if exists "Users can read shared readings" on public.readings;
drop policy if exists "Users can publish their profile" on public.profiles;
drop policy if exists "Users can update their profile" on public.profiles;
drop policy if exists "Users can read profiles" on public.profiles;
drop policy if exists "Users can manage their mates" on public.mates;
drop policy if exists "Users can see mate links involving them" on public.mates;

create policy "Users can manage their own readings"
on public.readings
for all
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);

create policy "Users can read shared readings"
on public.readings
for select
to authenticated
using (
  auth.uid() = user_id
  or exists (
    select 1
    from public.mates
    where (
      owner_id = auth.uid()
      and mate_id = readings.user_id
    )
    or (
      mate_id = auth.uid()
      and owner_id = readings.user_id
    )
  )
);

create policy "Users can publish their profile"
on public.profiles
for insert
to authenticated
with check (auth.uid() = id);

create policy "Users can update their profile"
on public.profiles
for update
to authenticated
using (auth.uid() = id)
with check (auth.uid() = id);

create policy "Users can read profiles"
on public.profiles
for select
to authenticated
using (true);

create policy "Users can manage their mates"
on public.mates
for all
to authenticated
using (auth.uid() = owner_id)
with check (auth.uid() = owner_id);

create policy "Users can see mate links involving them"
on public.mates
for select
to authenticated
using (auth.uid() = owner_id or auth.uid() = mate_id);
```

## Utilisation

1. Ouvrez la page dans votre navigateur.
2. Créez un compte ou connectez-vous.
3. Ajoutez ensuite vos relevés d'eau ou d'électricité.
4. Ouvrez les réglages pour ajouter un mate par e-mail et voir les relevés partagés.
