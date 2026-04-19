# State Management Patterns

## Overview
Client-side state management using Zustand with persistence and devtools support.

## Store Setup

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  error: string | null;
  setUser: (user: User, token: string) => void;
  clearUser: () => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      immer((set) => ({
        user: null,
        token: null,
        isLoading: false,
        error: null,
        setUser: (user, token) =>
          set((state) => {
            state.user = user;
            state.token = token;
            state.error = null;
          }),
        clearUser: () =>
          set((state) => {
            state.user = null;
            state.token = null;
          }),
        setLoading: (loading) =>
          set((state) => {
            state.isLoading = loading;
          }),
        setError: (error) =>
          set((state) => {
            state.error = error;
          }),
      })),
      { name: 'auth-storage', partialize: (s) => ({ token: s.token }) }
    ),
    { name: 'AuthStore' }
  )
);
```

## UI State (non-persistent)

```typescript
interface UIState {
  sidebarOpen: boolean;
  activeModal: string | null;
  toggleSidebar: () => void;
  openModal: (id: string) => void;
  closeModal: () => void;
}

export const useUIStore = create<UIState>()(devtools(
  (set) => ({
    sidebarOpen: true,
    activeModal: null,
    toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
    openModal: (id) => set({ activeModal: id }),
    closeModal: () => set({ activeModal: null }),
  }),
  { name: 'UIStore' }
));
```

## Selectors (avoid re-renders)

```typescript
// Derive stable selectors outside components
const selectUser = (s: AuthState) => s.user;
const selectIsAuthenticated = (s: AuthState) => s.user !== null && s.token !== null;

export function useIsAuthenticated() {
  return useAuthStore(selectIsAuthenticated);
}

export function useCurrentUser() {
  return useAuthStore(selectUser);
}
```

## Usage in Components

```typescript
export function ProfileButton() {
  const user = useCurrentUser();
  const clearUser = useAuthStore((s) => s.clearUser);

  if (!user) return null;

  return (
    <button onClick={clearUser}>
      {user.name}
    </button>
  );
}
```

## Guidelines
- Keep stores small and focused on a single domain
- Use `immer` for complex nested updates
- Use `persist` only for data that must survive page refresh
- Derive selectors outside components to prevent unnecessary re-renders
- Avoid storing server-fetched data in Zustand — use React Query for that
