# Logging Patterns

Structured logging utilities for consistent observability across the stack.

## Core Logger

```typescript
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  transport:
    process.env.NODE_ENV === 'development'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
  base: { service: process.env.SERVICE_NAME ?? 'app' },
  timestamp: pino.stdTimeFunctions.isoTime,
});

export function childLogger(context: Record<string, unknown>) {
  return logger.child(context);
}
```

## Request Logging Middleware (Express)

```typescript
import { Request, Response, NextFunction } from 'express';
import { v4 as uuid } from 'uuid';
import { childLogger } from './logger';

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const requestId = (req.headers['x-request-id'] as string) ?? uuid();
  const log = childLogger({ requestId, path: req.path, method: req.method });

  req.log = log;
  res.setHeader('x-request-id', requestId);

  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    const level = res.statusCode >= 500 ? 'error' : res.statusCode >= 400 ? 'warn' : 'info';
    log[level]({ statusCode: res.statusCode, duration }, 'request completed');
  });

  next();
}
```

## Typed Log Context

```typescript
interface LogContext {
  userId?: string;
  requestId?: string;
  traceId?: string;
  [key: string]: unknown;
}

export function withContext(ctx: LogContext) {
  return childLogger(ctx);
}
```

## Error Logging Helper

```typescript
import { logger } from './logger';

export function logError(err: unknown, context?: Record<string, unknown>) {
  if (err instanceof Error) {
    logger.error({ ...context, err: { message: err.message, stack: err.stack, name: err.name } }, err.message);
  } else {
    logger.error({ ...context, err }, 'unknown error');
  }
}
```

## React Hook — useLogger

```typescript
import { useMemo } from 'react';

interface ClientLogger {
  info(msg: string, data?: Record<string, unknown>): void;
  warn(msg: string, data?: Record<string, unknown>): void;
  error(msg: string, data?: Record<string, unknown>): void;
}

export function useLogger(component: string): ClientLogger {
  return useMemo(() => ({
    info: (msg, data) => console.info(`[${component}]`, msg, data ?? ''),
    warn: (msg, data) => console.warn(`[${component}]`, msg, data ?? ''),
    error: (msg, data) => console.error(`[${component}]`, msg, data ?? ''),
  }), [component]);
}
```

## Usage

```typescript
// Server
const log = withContext({ userId: user.id, requestId });
log.info('user logged in');
logError(err, { userId: user.id });

// React
const log = useLogger('UserProfile');
log.info('profile loaded', { userId });
```

## Notes
- Use `pino` on the server — fast, structured, JSON by default
- Attach `requestId` to every log within a request lifecycle
- Never log sensitive fields (passwords, tokens, PII)
- Use log levels consistently: `debug` for dev noise, `info` for business events, `warn` for recoverable issues, `error` for failures
