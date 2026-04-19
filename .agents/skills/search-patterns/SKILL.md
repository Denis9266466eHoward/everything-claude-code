# Search Patterns

Full-text search, filtering, and pagination patterns for backend and frontend.

## Backend: Search Service

```typescript
// services/search.ts
import { supabase } from '../lib/supabase';

export interface SearchParams {
  query: string;
  filters?: Record<string, unknown>;
  page?: number;
  limit?: number;
  orderBy?: string;
  orderDir?: 'asc' | 'desc';
}

export interface SearchResult<T> {
  data: T[];
  total: number;
  page: number;
  totalPages: number;
}

export async function searchUsers(
  params: SearchParams
): Promise<SearchResult<User>> {
  const { query, filters = {}, page = 1, limit = 20, orderBy = 'created_at', orderDir = 'desc' } = params;
  const offset = (page - 1) * limit;

  let builder = supabase
    .from('users')
    .select('*', { count: 'exact' })
    .or(`name.ilike.%${query}%,email.ilike.%${query}%`)
    .order(orderBy, { ascending: orderDir === 'asc' })
    .range(offset, offset + limit - 1);

  for (const [key, value] of Object.entries(filters)) {
    if (value !== undefined && value !== null) {
      builder = builder.eq(key, value);
    }
  }

  const { data, error, count } = await builder;
  if (error) throw error;

  return {
    data: data ?? [],
    total: count ?? 0,
    page,
    totalPages: Math.ceil((count ?? 0) / limit),
  };
}
```

## Frontend: useSearch Hook

```typescript
// hooks/useSearch.ts
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseSearchOptions<T> {
  fetcher: (query: string, page: number) => Promise<SearchResult<T>>;
  debounceMs?: number;
  initialQuery?: string;
}

export function useSearch<T>({ fetcher, debounceMs = 300, initialQuery = '' }: UseSearchOptions<T>) {
  const [query, setQuery] = useState(initialQuery);
  const [results, setResults] = useState<T[]>([]);
  const [total, setTotal] = useState(0);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const debounceRef = useRef<ReturnType<typeof setTimeout>>;

  const search = useCallback(async (q: string, p: number) => {
    setLoading(true);
    setError(null);
    try {
      const result = await fetcher(q, p);
      setResults(result.data);
      setTotal(result.total);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [fetcher]);

  useEffect(() => {
    clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      setPage(1);
      search(query, 1);
    }, debounceMs);
    return () => clearTimeout(debounceRef.current);
  }, [query, debounceMs, search]);

  useEffect(() => {
    if (page > 1) search(query, page);
  }, [page]);

  return { query, setQuery, results, total, page, setPage, loading, error };
}
```

## Usage

```typescript
const { query, setQuery, results, total, page, setPage, loading } = useSearch({
  fetcher: (q, p) => searchUsers({ query: q, page: p, limit: 20 }),
  debounceMs: 300,
});
```
