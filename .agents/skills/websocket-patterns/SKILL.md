# WebSocket Patterns

Real-time communication patterns using WebSockets with React and Node.js.

## Server Setup (Node.js/Express)

```typescript
import { WebSocketServer, WebSocket } from 'ws';
import { Server } from 'http';
import { verifyToken } from '../security';
import { logger } from '../logging';

interface Client {
  ws: WebSocket;
  userId: string;
  rooms: Set<string>;
}

const clients = new Map<string, Client>();

export function setupWebSocketServer(server: Server) {
  const wss = new WebSocketServer({ server, path: '/ws' });

  wss.on('connection', (ws, req) => {
    const token = new URL(req.url!, 'http://x').searchParams.get('token');
    if (!token) return ws.close(4001, 'Unauthorized');

    let userId: string;
    try {
      const payload = verifyToken(token);
      userId = payload.sub;
    } catch {
      return ws.close(4001, 'Invalid token');
    }

    const clientId = crypto.randomUUID();
    clients.set(clientId, { ws, userId, rooms: new Set() });
    logger.info({ userId, clientId }, 'ws client connected');

    ws.on('message', (data) => handleMessage(clientId, data));
    ws.on('close', () => {
      clients.delete(clientId);
      logger.info({ userId, clientId }, 'ws client disconnected');
    });
    ws.on('error', (err) => logger.error({ err, userId }, 'ws error'));

    send(ws, { type: 'connected', clientId });
  });

  return wss;
}

function handleMessage(clientId: string, data: any) {
  const client = clients.get(clientId);
  if (!client) return;

  try {
    const msg = JSON.parse(data.toString());
    if (msg.type === 'join') client.rooms.add(msg.room);
    if (msg.type === 'leave') client.rooms.delete(msg.room);
  } catch {
    logger.warn({ clientId }, 'invalid ws message');
  }
}

export function broadcast(room: string, payload: object) {
  for (const client of clients.values()) {
    if (client.rooms.has(room) && client.ws.readyState === WebSocket.OPEN) {
      send(client.ws, payload);
    }
  }
}

function send(ws: WebSocket, payload: object) {
  ws.send(JSON.stringify(payload));
}
```

## React Hook

```typescript
import { useEffect, useRef, useCallback, useState } from 'react';

interface UseWebSocketOptions {
  token: string;
  onMessage?: (msg: any) => void;
}

export function useWebSocket({ token, onMessage }: UseWebSocketOptions) {
  const ws = useRef<WebSocket | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const url = `${location.origin.replace('http', 'ws')}/ws?token=${token}`;
    ws.current = new WebSocket(url);

    ws.current.onopen = () => setConnected(true);
    ws.current.onclose = () => setConnected(false);
    ws.current.onmessage = (e) => {
      try { onMessage?.(JSON.parse(e.data)); } catch {}
    };

    return () => ws.current?.close();
  }, [token]);

  const send = useCallback((payload: object) => {
    if (ws.current?.readyState === WebSocket.OPEN) {
      ws.current.send(JSON.stringify(payload));
    }
  }, []);

  const join = useCallback((room: string) => send({ type: 'join', room }), [send]);
  const leave = useCallback((room: string) => send({ type: 'leave', room }), [send]);

  return { connected, send, join, leave };
}
```

## Usage

```typescript
// Server: broadcast when data changes
broadcast('market:BTC', { type: 'price', data: { price: 42000 } });

// Client
const { connected, join } = useWebSocket({
  token,
  onMessage: (msg) => {
    if (msg.type === 'price') updatePrice(msg.data);
  },
});
useEffect(() => { if (connected) join('market:BTC'); }, [connected]);
```
