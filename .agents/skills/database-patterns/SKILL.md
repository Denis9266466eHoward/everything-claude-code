# Database Patterns

Skills for working with databases, migrations, and query optimization.

## Supabase / PostgreSQL Patterns

### Schema Design

```sql
-- Always use UUIDs for primary keys
create table users (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Enable RLS on all tables
alter table users enable row level security;

-- Trigger to auto-update updated_at
create or replace function update_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger users_updated_at
  before update on users
  for each row execute function update_updated_at();
```

### Row Level Security

```sql
-- Users can only read/write their own data
create policy "users_select_own" on users
  for select using (auth.uid() = id);

create policy "users_update_own" on users
  for update using (auth.uid() = id);
```

### TypeScript Query Patterns

```typescript
import { createClient } from '@supabase/supabase-js';
import type { Database } from './database.types';

const supabase = createClient<Database>(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!
);

// Typed query helper
async function getUser(id: string) {
  const { data, error } = await supabase
    .from('users')
    .select('id, email, created_at')
    .eq('id', id)
    .single();

  if (error) throw new Error(error.message);
  return data;
}

// Paginated query
async function listUsers(page = 1, pageSize = 20) {
  const from = (page - 1) * pageSize;
  const to = from + pageSize - 1;

  const { data, error, count } = await supabase
    .from('users')
    .select('*', { count: 'exact' })
    .range(from, to)
    .order('created_at', { ascending: false });

  if (error) throw new Error(error.message);
  return { data, total: count, page, pageSize };
}
```

### Migrations

```sql
-- migrations/001_create_users.sql
create table if not exists users (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  full_name text,
  avatar_url text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- migrations/002_add_user_roles.sql
create type user_role as enum ('admin', 'member', 'viewer');

alter table users
  add column role user_role not null default 'member';

create index users_role_idx on users(role);
```

### Realtime Subscriptions

```typescript
function subscribeToUserChanges(userId: string, onUpdate: (user: User) => void) {
  const channel = supabase
    .channel(`user:${userId}`)
    .on(
      'postgres_changes',
      { event: 'UPDATE', schema: 'public', table: 'users', filter: `id=eq.${userId}` },
      (payload) => onUpdate(payload.new as User)
    )
    .subscribe();

  return () => supabase.removeChannel(channel);
}
```

## Query Optimization Tips

- Always add indexes for columns used in WHERE/JOIN clauses
- Use `select('col1, col2')` instead of `select('*')` to reduce payload
- Prefer server-side filtering over client-side filtering
- Use database functions for complex business logic to reduce round trips
