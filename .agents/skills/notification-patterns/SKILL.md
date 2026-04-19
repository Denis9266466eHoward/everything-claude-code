# Notification Patterns

Skill for implementing push notifications, in-app alerts, and email notifications.

## Backend: Notification Service

```typescript
// services/notifications.ts
import { supabase } from '../lib/supabase';
import { logger } from '../lib/logger';

export type NotificationType = 'info' | 'success' | 'warning' | 'error';

export interface Notification {
  id: string;
  userId: string;
  type: NotificationType;
  title: string;
  message: string;
  read: boolean;
  createdAt: string;
  metadata?: Record<string, unknown>;
}

export async function createNotification(
  userId: string,
  type: NotificationType,
  title: string,
  message: string,
  metadata?: Record<string, unknown>
): Promise<Notification> {
  const { data, error } = await supabase
    .from('notifications')
    .insert({ user_id: userId, type, title, message, metadata })
    .select()
    .single();

  if (error) throw error;
  logger.info({ userId, type, title }, 'notification created');
  return data;
}

export async function markAsRead(notificationId: string, userId: string): Promise<void> {
  const { error } = await supabase
    .from('notifications')
    .update({ read: true })
    .eq('id', notificationId)
    .eq('user_id', userId);

  if (error) throw error;
}

export async function markAllAsRead(userId: string): Promise<void> {
  const { error } = await supabase
    .from('notifications')
    .update({ read: true })
    .eq('user_id', userId)
    .eq('read', false);

  if (error) throw error;
}
```

## Frontend: useNotifications Hook

```typescript
// hooks/useNotifications.ts
import { useEffect, useState, useCallback } from 'react';
import { supabase } from '../lib/supabase';
import type { Notification } from '../services/notifications';

export function useNotifications(userId: string) {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [unreadCount, setUnreadCount] = useState(0);
  const [loading, setLoading] = useState(true);

  const fetchNotifications = useCallback(async () => {
    const { data } = await supabase
      .from('notifications')
      .select('*')
      .eq('user_id', userId)
      .order('created_at', { ascending: false })
      .limit(50);

    if (data) {
      setNotifications(data);
      setUnreadCount(data.filter((n) => !n.read).length);
    }
    setLoading(false);
  }, [userId]);

  useEffect(() => {
    fetchNotifications();

    // Subscribe to realtime inserts
    const channel = supabase
      .channel(`notifications:${userId}`)
      .on(
        'postgres_changes',
        { event: 'INSERT', schema: 'public', table: 'notifications', filter: `user_id=eq.${userId}` },
        (payload) => {
          setNotifications((prev) => [payload.new as Notification, ...prev]);
          setUnreadCount((c) => c + 1);
        }
      )
      .subscribe();

    return () => { supabase.removeChannel(channel); };
  }, [userId, fetchNotifications]);

  const markRead = useCallback(async (id: string) => {
    await supabase.from('notifications').update({ read: true }).eq('id', id);
    setNotifications((prev) => prev.map((n) => n.id === id ? { ...n, read: true } : n));
    setUnreadCount((c) => Math.max(0, c - 1));
  }, []);

  const markAllRead = useCallback(async () => {
    await supabase.from('notifications').update({ read: true }).eq('user_id', userId).eq('read', false);
    setNotifications((prev) => prev.map((n) => ({ ...n, read: true })));
    setUnreadCount(0);
  }, [userId]);

  return { notifications, unreadCount, loading, markRead, markAllRead, refetch: fetchNotifications };
}
```

## Frontend: NotificationBell Component

```tsx
// components/NotificationBell.tsx
import { useState } from 'react';
import { useNotifications } from '../hooks/useNotifications';
import { useCurrentUser } from '../hooks/useAuth';

export function NotificationBell() {
  const user = useCurrentUser();
  const { notifications, unreadCount, markRead, markAllRead } = useNotifications(user?.id ?? '');
  const [open, setOpen] = useState(false);

  return (
    <div className="relative">
      <button onClick={() => setOpen((o) => !o)} className="relative p-2">
        <span>🔔</span>
        {unreadCount > 0 && (
          <span className="absolute top-0 right-0 bg-red-500 text-white text-xs rounded-full px-1">
            {unreadCount > 99 ? '99+' : unreadCount}
          </span>
        )}
      </button>

      {open && (
        <div className="absolute right-0 mt-2 w-80 bg-white shadow-lg rounded-lg z-50">
          <div className="flex justify-between items-center p-3 border-b">
            <span className="font-semibold">Notifications</span>
            {unreadCount > 0 && (
              <button onClick={markAllRead} className="text-sm text-blue-500">Mark all read</button>
            )}
          </div>
          <ul className="max-h-96 overflow-y-auto">
            {notifications.length === 0 && (
              <li className="p-4 text-center text-gray-400">No notifications</li>
            )}
            {notifications.map((n) => (
              <li
                key={n.id}
                onClick={() => markRead(n.id)}
                className={`p-3 border-b cursor-pointer hover:bg-gray-50 ${
                  !n.read ? 'bg-blue-50' : ''
                }`}
              >
                <p className="font-medium text-sm">{n.title}</p>
                <p className="text-xs text-gray-500">{n.message}</p>
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
}
```

## Usage

```typescript
// Send a notification from backend
await createNotification(
  userId,
  'success',
  'Order Confirmed',
  'Your order #1234 has been confirmed.',
  { orderId: '1234' }
);
```
