# Frontend Patterns Skill

Expert knowledge of modern frontend patterns, React best practices, state management, and component architecture.

## Core Competencies

- React component patterns (compound, render props, HOC, hooks)
- State management (Zustand, Jotai, Redux Toolkit)
- Data fetching patterns (React Query, SWR)
- Performance optimization (memoization, code splitting, virtualization)
- TypeScript integration

## Common Patterns

### Custom Hook Pattern

```typescript
// useLocalStorage.ts
import { useState, useEffect } from 'react';

function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  const setValue = (value: T | ((valToStore = value instanceof Function ? valueToStore);
      window.{
      console.error(error);
    }
  };

  return [storedValue, setValue] as const;
}
```

### React Query Data Fetching

```typescript
// hooks/useUsers.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api';

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => api.get('/users').then(res => res.data),
    staleTime: 5 * 60 * 1000,
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: CreateUserDto) => api.post('/users', data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

### Zustand Store Pattern

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AuthState {
  user: User | null;
  token: string | null;
  setUser: (user: User, token: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      setUser: (user, token) => set({ user, token }),
      logout: () => set({ user: null, token: null }),
    }),
    { name: 'auth-storage' }
  )
);
```

### Compound Component Pattern

```typescript
// components/Card/index.tsx
import React, { createContext, useContext } from 'react';

const CardContext = createContext<{ variant: string }>({ variant: 'default' });

function Card({ children, variant = 'default' }: CardProps) {
  return (
    <CardContext.Provider value={{ variant }}>
      <div className={`card card--${variant}`}>{children}</div>
    </CardContext.Provider>
  );
}

Card.Header = function CardHeader({ children }: { children: React.ReactNode }) {
  const { variant } = useContext(CardContext);
  return <div className={`card__header card__header--${variant}`}>{children}</div>;
};

Card.Body = function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card__body">{children}</div>;
};

export { Card };
```

## Performance Patterns

- Use `React.memo` for expensive pure components
- Use `useMemo` for expensive computations, not for object identity
- Use `useCallback` when passing callbacks to memoized children
- Prefer `useTransition` for non-urgent state updates
- Lazy load routes with `React.lazy` and `Suspense`

## Anti-Patterns to Avoid

- Prop drilling beyond 2-3 levels (use context or state management)
- Storing derived state (compute from source of truth)
- Over-fetching with useEffect + useState (use React Query)
- Premature memoization (profile first)
