# Caching Patterns

Strategies for caching data on the backend and frontend.

## Backend: Redis Cache

```typescript
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

export async function getOrSet<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds = 60
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached) as T;

  const value = await fetcher();
  await redis.setEx(key, ttlSeconds, JSON.stringify(value));
  return value;
}

export async function invalidate(key: string) {
  await redis.del(key);
}

export async function invalidatePattern(pattern: string) {
  const keys = await redis.keys(pattern);
  if (keys.length) await redis.del(keys);
}
```

## Cache Key Helpers

```typescript
export const cacheKeys = {
  user: (id: string) => `user:${id}`,
  userList: () => `users:list`,
  market: (id: string) => `market:${id}`,
};
```

## Express Middleware

```typescript
import { Request, Response, NextFunction } from 'express';

export function cacheMiddleware(ttlSeconds = 60) {
  return async (req: Request, res: Response, next: NextFunction) => {
    if (req.method !== 'GET') return next();

    const key = `http:${req.originalUrl}`;
    const cached = await redis.get(key);
    if (cached) {
      res.setHeader('X-Cache', 'HIT');
      return res.json(JSON.parse(cached));
    }

    const originalJson = res.json.bind(res);
    res.json = (body) => {
      redis.setEx(key, ttlSeconds, JSON.stringify(body));
      res.setHeader('X-Cache', 'MISS');
      return originalJson(body);
    };

    next();
  };
}
```

## Frontend: React Query Config

```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,      // 1 minute
      gcTime: 1000 * 60 * 10,    // 10 minutes
      retry: 2,
      refetchOnWindowFocus: false,
    },
  },
});

// Invalidate after mutation
export function useInvalidate() {
  return (keys: string[]) => queryClient.invalidateQueries({ queryKey: keys });
}
```

## In-Memory LRU (no Redis)

```typescript
import LRU from 'lru-cache';

const lru = new LRU<string, unknown>({ max: 500, ttl: 1000 * 60 });

export function lruGetOrSet<T>(key: string, fetcher: () => T): T {
  if (lru.has(key)) return lru.get(key) as T;
  const value = fetcher();
  lru.set(key, value);
  return value;
}
```

## Notes

- Prefer Redis for shared/distributed caches across multiple instances
- Use React Query for client-side server-state caching
- Always invalidate on write (create/update/delete)
- Use pattern-based invalidation (`users:*`) for list caches
- LRU is fine for single-instance or dev environments
