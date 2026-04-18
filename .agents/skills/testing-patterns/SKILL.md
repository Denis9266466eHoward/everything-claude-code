# Testing Patterns

Patterns and best practices for writing tests in JavaScript/TypeScript projects.

## Unit Testing with Vitest

```typescript
// src/lib/__tests__/marketService.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { MarketService } from '../marketService'
import type { MarketRepository } from '../types'

const mockRepo: MarketRepository = {
  findAll: vi.fn(),
  findById: vi.fn(),
  create: vi.fn(),
  update: vi.fn(),
  delete: vi.fn(),
}

describe('MarketService', () => {
  let service: MarketService

  beforeEach(() => {
    vi.clearAllMocks()
    service = new MarketService(mockRepo)
  })

  it('returns all markets', async () => {
    const markets = [{ id: '1', name: 'Test Market' }]
    vi.mocked(mockRepo.findAll).mockResolvedValue(markets)

    const result = await service.getMarkets()
    expect(result).toEqual(markets)
    expect(mockRepo.findAll).toHaveBeenCalledOnce()
  })

  it('throws when market not found', async () => {
    vi.mocked(mockRepo.findById).mockResolvedValue(null)
    await expect(service.getMarketById('missing')).rejects.toThrow('Market not found')
  })
})
```

## React Component Testing with Testing Library

```typescript
// src/components/__tests__/Card.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Card } from '../Card'

describe('Card', () => {
  it('renders title and children', () => {
    render(<Card title="Hello"><p>Content</p></Card>)
    expect(screen.getByText('Hello')).toBeInTheDocument()
    expect(screen.getByText('Content')).toBeInTheDocument()
  })

  it('calls onClick when clicked', async () => {
    const user = userEvent.setup()
    const onClick = vi.fn()
    render(<Card title="Clickable" onClick={onClick}>body</Card>)
    await user.click(screen.getByRole('article'))
    expect(onClick).toHaveBeenCalledOnce()
  })
})
```

## Hook Testing

```typescript
// src/hooks/__tests__/useLocalStorage.test.ts
import { renderHook, act } from '@testing-library/react'
import { useLocalStorage } from '../useLocalStorage'

describe('useLocalStorage', () => {
  beforeEach(() => localStorage.clear())

  it('returns initial value when key not set', () => {
    const { result } = renderHook(() => useLocalStorage('key', 'default'))
    expect(result.current[0]).toBe('default')
  })

  it('persists value to localStorage', () => {
    const { result } = renderHook(() => useLocalStorage('key', ''))
    act(() => result.current[1]('saved'))
    expect(localStorage.getItem('key')).toBe(JSON.stringify('saved'))
  })
})
```

## MSW for API Mocking

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'Alice', email: 'alice@example.com' },
    ])
  }),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: '2', ...body }, { status: 201 })
  }),
]
```

## Guidelines

- Test behavior, not implementation
- One assertion concept per test
- Use `beforeEach` to reset mocks and state
- Prefer `userEvent` over `fireEvent` for interactions
- Mock at the boundary (API layer), not deep internals
