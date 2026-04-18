# Performance Patterns

Skills for optimizing frontend and backend performance.

## Frontend Performance

### React Memoization

```tsx
import { memo, useMemo, useCallback } from 'react';

// Memoize expensive component renders
const UserList = memo(({ users, onSelect }: UserListProps) => {
  return (
    <ul>
      {users.map(user => (
        <UserItem key={user.id} user={user} onSelect={onSelect} />
      ))}
    </ul>
  );
});

// Memoize derived data
function useFilteredUsers(users: User[], query: string) {
  return useMemo(
    () => users.filter(u => u.name.toLowerCase().includes(query.toLowerCase())),
    [users, query]
  );
}

// Stable callback references
function useUserActions(userId: string) {
  const deleteUser = useCallback(async () => {
    await api.delete(`/users/${userId}`);
  }, [userId]);

  return { deleteUser };
}
```

### Virtual Scrolling

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: virtualItem.start,
              height: virtualItem.size,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Backend Performance

### Request Batching & Debouncing

```ts
// Batch multiple DB lookups into a single query
class UserLoader {
  private queue: string[] = [];
  private timer: ReturnType<typeof setTimeout> | null = null;
  private resolvers = new Map<string, (user: User) => void>();

  load(id: string): Promise<User> {
    return new Promise(resolve => {
      this.resolvers.set(id, resolve);
      this.queue.push(id);
      this.scheduleFlush();
    });
  }

  private scheduleFlush() {
    if (this.timer) return;
    this.timer = setTimeout(() => this.flush(), 0);
  }

  private async flush() {
    const ids = [...this.queue];
    this.queue = [];
    this.timer = null;

    const users = await db.users.findMany({ where: { id: { in: ids } } });
    users.forEach(user => this.resolvers.get(user.id)?.(user));
    this.resolvers.clear();
  }
}
```

### Response Caching

```ts
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

export function cached<T>(ttlSeconds: number) {
  return function (target: object, key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = async function (...args: unknown[]) {
      const cacheKey = `${key}:${JSON.stringify(args)}`;
      const cached = await redis.get(cacheKey);
      if (cached) return JSON.parse(cached) as T;

      const result = await original.apply(this, args);
      await redis.setex(cacheKey, ttlSeconds, JSON.stringify(result));
      return result;
    };
    return descriptor;
  };
}

// Usage
class ProductService {
  @cached(300)
  async getProduct(id: string): Promise<Product> {
    return db.products.findUniqueOrThrow({ where: { id } });
  }
}
```

## Tips

- Use `React.lazy` + `Suspense` for code splitting
- Prefer `select` in Prisma/Supabase to fetch only needed fields
- Add database indexes on frequently queried columns
- Use `stale-while-revalidate` cache headers for API responses
- Profile before optimizing — measure, don't guess
