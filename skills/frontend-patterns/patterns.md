# Skill: Frontend Patterns

Reusable patterns for UI components, state management, data fetching, and performance.

---

## Component Patterns

### Presentational vs Container Components

```typescript
// Presentational — pure UI, no data fetching
interface OrderCardProps {
  order: Order
  onCancel: (id: string) => void
}
function OrderCard({ order, onCancel }: OrderCardProps) {
  return (
    <article className="order-card">
      <h3>Order #{order.id}</h3>
      <p>Status: {order.status}</p>
      <button onClick={() => onCancel(order.id)}>Cancel</button>
    </article>
  )
}

// Container — handles data, delegates to presentational
function OrderCardContainer({ orderId }: { orderId: string }) {
  const { data: order } = useOrder(orderId)
  const { mutate: cancelOrder } = useCancelOrder()

  if (!order) return <OrderCardSkeleton />
  return <OrderCard order={order} onCancel={cancelOrder} />
}
```

### Compound Components
```typescript
// Compose complex UI from coordinated sub-components
<Tabs defaultValue="overview">
  <Tabs.List>
    <Tabs.Trigger value="overview">Overview</Tabs.Trigger>
    <Tabs.Trigger value="details">Details</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="overview"><Overview /></Tabs.Content>
  <Tabs.Content value="details"><Details /></Tabs.Content>
</Tabs>
```

### Render Props / Children as Function
```typescript
function DataFetcher<T>({ url, children }: {
  url: string
  children: (state: AsyncState<T>) => React.ReactNode
}) {
  const state = useFetch<T>(url)
  return <>{children(state)}</>
}

// Usage
<DataFetcher url="/api/orders">
  {({ data, isLoading, error }) => (
    isLoading ? <Spinner /> : error ? <Error /> : <OrderList orders={data!} />
  )}
</DataFetcher>
```

---

## State Management Patterns

### Server State (TanStack Query / SWR)

```typescript
// Define queries with explicit keys
const orderKeys = {
  all: ['orders'] as const,
  list: (filters: OrderFilters) => [...orderKeys.all, 'list', filters] as const,
  detail: (id: string) => [...orderKeys.all, 'detail', id] as const,
}

// Query hook
function useOrder(id: string) {
  return useQuery({
    queryKey: orderKeys.detail(id),
    queryFn: () => api.orders.get(id),
    staleTime: 30_000,
    retry: (failureCount, error) => failureCount < 3 && !isNotFoundError(error),
  })
}

// Mutation with optimistic update
function useCancelOrder() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: api.orders.cancel,
    onMutate: async (orderId) => {
      await queryClient.cancelQueries({ queryKey: orderKeys.detail(orderId) })
      const previous = queryClient.getQueryData(orderKeys.detail(orderId))
      queryClient.setQueryData(orderKeys.detail(orderId), (old: Order) => ({
        ...old, status: 'cancelling'
      }))
      return { previous }
    },
    onError: (err, orderId, ctx) => {
      queryClient.setQueryData(orderKeys.detail(orderId), ctx?.previous)
    },
    onSettled: (_, __, orderId) => {
      queryClient.invalidateQueries({ queryKey: orderKeys.detail(orderId) })
    },
  })
}
```

### Global Client State (Zustand)

```typescript
interface UIStore {
  sidebarOpen: boolean
  theme: 'light' | 'dark'
  toggleSidebar: () => void
  setTheme: (theme: 'light' | 'dark') => void
}

const useUIStore = create<UIStore>((set) => ({
  sidebarOpen: false,
  theme: 'light',
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
  setTheme: (theme) => set({ theme }),
}))
```

---

## Error Handling Patterns

```typescript
// Error Boundary + Suspense
function OrderPage({ id }: { id: string }) {
  return (
    <ErrorBoundary fallback={<ErrorMessage />} onError={logError}>
      <Suspense fallback={<OrderPageSkeleton />}>
        <OrderPageContent id={id} />
      </Suspense>
    </ErrorBoundary>
  )
}

// Typed error handling in queries
type ApiError = { code: string; message: string }

function useOrder(id: string) {
  return useQuery<Order, ApiError>({
    queryKey: ['order', id],
    queryFn: async () => {
      const res = await fetch(`/api/orders/${id}`)
      if (!res.ok) throw await res.json() as ApiError
      return res.json()
    },
  })
}
```

---

## Form Patterns (React Hook Form + Zod)

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email: z.string().email('Please enter a valid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})

type LoginForm = z.infer<typeof schema>

function LoginForm({ onSubmit }: { onSubmit: (data: LoginForm) => Promise<void> }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<LoginForm>({
    resolver: zodResolver(schema),
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')}
          aria-describedby={errors.email ? 'email-error' : undefined}
          aria-invalid={!!errors.email} />
        {errors.email && (
          <p id="email-error" role="alert">{errors.email.message}</p>
        )}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </button>
    </form>
  )
}
```

---

## Custom Hooks Patterns

```typescript
// Encapsulate complex logic in a custom hook
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key)
      return item ? JSON.parse(item) : initialValue
    } catch {
      return initialValue
    }
  })

  const setStoredValue = useCallback((newValue: T | ((prev: T) => T)) => {
    setValue((prev) => {
      const resolved = newValue instanceof Function ? newValue(prev) : newValue
      window.localStorage.setItem(key, JSON.stringify(resolved))
      return resolved
    })
  }, [key])

  return [value, setStoredValue] as const
}
```

---

## Accessibility Patterns

```typescript
// Focus trap for modals
function Modal({ isOpen, onClose, children }: ModalProps) {
  const ref = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (isOpen) ref.current?.focus()
  }, [isOpen])

  if (!isOpen) return null
  return (
    <div
      ref={ref}
      role="dialog"
      aria-modal="true"
      tabIndex={-1}
      onKeyDown={(e) => e.key === 'Escape' && onClose()}
    >
      <button onClick={onClose} aria-label="Close modal">×</button>
      {children}
    </div>
  )
}

// Announce dynamic content to screen readers
function StatusAnnouncer({ message }: { message: string }) {
  return (
    <div aria-live="polite" aria-atomic="true" className="sr-only">
      {message}
    </div>
  )
}
```

---

## Performance Patterns

```typescript
// Lazy load heavy dependencies
const RichTextEditor = React.lazy(() =>
  import('./RichTextEditor').then((m) => ({ default: m.RichTextEditor }))
)

// Virtualise long lists
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null)
  const virtualiser = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
  })

  return (
    <div ref={parentRef} style={{ height: '500px', overflow: 'auto' }}>
      <div style={{ height: `${virtualiser.getTotalSize()}px`, position: 'relative' }}>
        {virtualiser.getVirtualItems().map((row) => (
          <div key={row.key} style={{ position: 'absolute', top: row.start, width: '100%' }}>
            <ItemRow item={items[row.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```
