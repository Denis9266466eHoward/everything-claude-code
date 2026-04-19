# Rate Limiting Patterns

Skill for implementing rate limiting on APIs and services.

## Backend: Express Rate Limiter

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../lib/redis';
import { ApiError } from '../lib/errors';

// Basic rate limiter
export const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  handler: (_req, _res, next) => {
    next(new ApiError(429, 'Too many requests, please try again later'));
  },
});

// Stricter limiter for auth endpoints
export const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10,
  standardHeaders: true,
  legacyHeaders: false,
  handler: (_req, _res, next) => {
    next(new ApiError(429, 'Too many login attempts, please try again later'));
  },
});

// Redis-backed limiter for distributed environments
export const distributedLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.sendCommand(args),
  }),
  handler: (_req, _res, next) => {
    next(new ApiError(429, 'Too many requests'));
  },
});

// Per-user rate limiter using key generator
export const userRateLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 30,
  keyGenerator: (req) => req.user?.id ?? req.ip,
  handler: (_req, _res, next) => {
    next(new ApiError(429, 'Rate limit exceeded'));
  },
});
```

## Backend: Token Bucket (custom)

```typescript
import { redis } from '../lib/redis';

/**
 * Token bucket rate limiter backed by Redis.
 * Allows bursting up to `capacity` then refills at `refillRate` tokens/sec.
 */
export async function tokenBucket(
  key: string,
  capacity: number,
  refillRate: number
): Promise<{ allowed: boolean; remaining: number }> {
  const now = Date.now() / 1000;
  const bucketKey = `bucket:${key}`;

  const result = await redis.eval(
    `
    local tokens = tonumber(redis.call('hget', KEYS[1], 'tokens') or ARGV[1])
    local last = tonumber(redis.call('hget', KEYS[1], 'last') or ARGV[3])
    local capacity = tonumber(ARGV[1])
    local rate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    local delta = math.max(0, now - last)
    tokens = math.min(capacity, tokens + delta * rate)
    local allowed = tokens >= 1
    if allowed then tokens = tokens - 1 end
    redis.call('hset', KEYS[1], 'tokens', tokens, 'last', now)
    redis.call('expire', KEYS[1], math.ceil(capacity / rate) + 1)
    return { allowed and 1 or 0, math.floor(tokens) }
    `,
    1,
    bucketKey,
    capacity,
    refillRate,
    now
  ) as [number, number];

  return { allowed: result[0] === 1, remaining: result[1] };
}
```

## Usage

```typescript
// Apply globally
app.use(rateLimiter);

// Apply to specific routes
app.post('/auth/login', authLimiter, loginHandler);
app.post('/auth/register', authLimiter, registerHandler);

// Per-user on sensitive endpoints
app.post('/api/messages', userRateLimiter, sendMessageHandler);

// Custom token bucket
app.post('/api/export', async (req, res, next) => {
  const { allowed, remaining } = await tokenBucket(`export:${req.user.id}`, 5, 0.1);
  if (!allowed) return next(new ApiError(429, 'Export rate limit exceeded'));
  res.setHeader('X-RateLimit-Remaining', remaining);
  next();
});
```
