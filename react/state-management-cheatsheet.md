# State Management Cheatsheet

Distilled from Steve Kinney's [React State Management course](https://stevekinney.com/courses/react-state). Patterns use ✅ for recommended and ❌ for anti-patterns.

---

## 1. Cardinal Principles

1. **State is a snapshot derived from events.** Events capture intent; state is just a projection of that history.
2. **Make impossible states impossible.** Discriminated unions beat boolean soup (`isLoading && isError && isSuccess`).
3. **Pure functions for app logic.** Reducers, selectors, validators: testable, replayable, framework-agnostic.
4. **Match the tool to the state's lifetime and scope.** Render-time derivation → ref → `useState` → URL → server-state library → external store. Smallest tool that fits.
5. **One `useEffect`, not many.** Effects synchronise with the outside world; they don't orchestrate internal transitions.

---

## 2. What NOT to put in `useState`

### Derived state: calculate, don't store

```tsx
// ❌ Two sources of truth, sync via effect
const [items, setItems] = useState<Item[]>([]);
const [total, setTotal] = useState(0);
useEffect(() => { setTotal(items.reduce((s, i) => s + i.cost, 0)); }, [items]);

// ✅ Derive in render
const total = items.reduce((s, i) => s + i.cost, 0);
```

Reach for `useMemo` only if the calculation is genuinely expensive. Most aren't.

### Refs for values the UI doesn't render

`useState` triggers a re-render. `useRef` doesn't. Use a ref for timer ids, scroll positions, previous values, anything that survives between renders but isn't displayed.

```tsx
const timerIdRef = useRef<number | null>(null);

const start = () => {
  if (timerIdRef.current) clearInterval(timerIdRef.current);
  timerIdRef.current = window.setInterval(tick, 1000);
};

useEffect(() => () => {
  if (timerIdRef.current) clearInterval(timerIdRef.current);
}, []);
```

### Store ids, derive objects

```tsx
// ✅ Look up by id at render time
const selected = users.find((u) => u.id === selectedUserId);

// ❌ Two copies of the same object that can drift
const [selectedUser, setSelectedUser] = useState<User | null>(null);
```

### Don't duplicate props or context into state

If a value lives in props or context, render against it directly. Copying it into `useState` makes it stale the moment the source changes.

---

## 3. Modelling Before Coding

Before writing any state code, write a plain-text `flows.md` in version control. Three diagrams cover most cases.

**Entity-Relationship** (what exists, how it relates):

```
- User    { id, name }
- Cart    { id, userId }
- Item    { id, cartId, sku, qty }
```

**Sequence** (who calls whom, in order):

```
UI    -> API   : addItem(sku)
API   -> DB    : persist
API   -> UI    : updatedCart
UI    -> UI    : render
```

**State** (every state, its data, its outgoing events):

```
idle
  stores: {}
  on submit -> loading

loading
  effect: fetch
  on success -> ready
  on failure -> error

ready
  stores: { data }
```

Modelling separates *incidental* complexity (irreducible) from *accidental* complexity (self-inflicted). Done in 10 minutes; saves hours.

---

## 4. Discriminated Unions Over Booleans

Replace many `useState` calls and boolean flags with one state object plus a discriminated union for status. Booleans silently allow impossible combinations; a single `status` literal makes those states unreachable.

```tsx
// ❌ Boolean soup: what does isLoading=true, error="oops", data=[...] mean?
const [isLoading, setIsLoading] = useState(false);
const [error, setError]         = useState<string | null>(null);
const [data, setData]           = useState<User[] | null>(null);

// ✅ One discriminated union: each variant carries exactly what it needs
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };

const [state, setState] = useState<FetchState<User[]>>({ status: 'idle' });

// `data` is only accessible when status === 'success': TS enforces it
if (state.status === 'success') return <List users={state.data} />;
```

Submit becomes a single atomic transition instead of three separate `set*` calls:

```tsx
const submit = async () => {
  setState({ status: 'loading' });
  try {
    const data = await fetchUsers();
    setState({ status: 'success', data });
  } catch (e) {
    setState({ status: 'error', message: String(e) });
  }
};
```

---

## 5. useReducer + Context

Once state has structure (events, transitions, multiple consumers), `useState` stops scaling. `useReducer` puts all transitions in one pure function. Context plus a custom hook eliminates prop drilling. The reducer **is** the state machine.

### Action type describes intent

```tsx
type CartAction =
  | { type: 'add';    sku: string }
  | { type: 'remove'; sku: string }
  | { type: 'clear' };
```

### Reducer

```tsx
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add':    return { ...state, items: [...state.items, { sku: action.sku, qty: 1 }] };
    case 'remove': return { ...state, items: state.items.filter((i) => i.sku !== action.sku) };
    case 'clear':  return { ...state, items: [] };
    default:
      const _: never = action; // ✅ compile error if a case is missed
      return state;
  }
}
```

### Provider + custom hook

```tsx
const CartContext = createContext<{
  state: CartState;
  dispatch: (a: CartAction) => void;
} | null>(null);

export function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, initialCart);
  return <CartContext.Provider value={{ state, dispatch }}>{children}</CartContext.Provider>;
}

export function useCart() {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error('useCart must be used inside CartProvider');
  return ctx;
}
```

### React 19: `use()` replaces `useContext`

```tsx
const { state, dispatch } = use(CartContext);
```

```tsx
// ❌ Prop-drilling state/setState through three layers that just relay
<Page state={state} setState={setState}>
  <Form state={state} setState={setState}>
    <Field state={state} setState={setState} />

// ✅ Lift into context once
<CartProvider><Page /></CartProvider>
```

If you type the same prop more than twice, lift it.

---

## 6. Forms: FormData, Server Actions, Zod

Controlled inputs with one `useState` per field is boilerplate that grows linearly. The platform already gives you `FormData`. Next.js server actions submit directly to a server function; `useActionState` handles pending and response state; Zod validates server-side. Degrades gracefully without JavaScript.

```tsx
const initial: FormState = { status: 'idle', errors: {}, data: null };

export default function SignupPage() {
  const [state, action, isPending] = useActionState(submitSignup, initial);

  if (state.status === 'success') return <Success data={state.data} />;

  return (
    <form action={action} className="space-y-6">
      <Input
        name="email"
        defaultValue={state.submitted?.email ?? ''}
        aria-invalid={!!state.errors?.email}
        disabled={isPending}
      />
      {state.errors?.email && <p className="text-red-600">{state.errors.email}</p>}
      <Button type="submit" disabled={isPending}>{isPending ? 'Submitting…' : 'Sign up'}</Button>
    </form>
  );
}
```

`name="email"` plus `defaultValue` (uncontrolled). `FormData` picks fields up by name.

```tsx
'use server';

const schema = z.object({
  email: z.string().email(),
  name:  z.string().min(1),
});

export async function submitSignup(prev: FormState, formData: FormData): Promise<FormState> {
  const result = schema.safeParse(Object.fromEntries(formData));
  if (!result.success) {
    return {
      status: 'error',
      errors: result.error.flatten().fieldErrors,
      submitted: Object.fromEntries(formData) as Partial<Signup>,
    };
  }
  await save(result.data);
  return { status: 'success', data: result.data, errors: {} };
}
```

Zod doubles as runtime validation and TS type: `type Signup = z.infer<typeof schema>`.

```tsx
// ❌ One useState per field, per error, plus a hand-rolled isSubmitting flag.
// You're reinventing the form element.
```

---

## 7. URL State: `nuqs`

State that should be shareable, bookmarkable, or survive a refresh belongs in the URL, not `useState`. Search filters, pagination, tabs, form drafts: all URL state.

```tsx
import { useQueryState, parseAsBoolean, parseAsInteger, parseAsStringEnum } from 'nuqs';

function Filters() {
  const [q]         = useQueryState('q');
  const [page]      = useQueryState('page',   parseAsInteger.withDefault(1));
  const [activeOnly]= useQueryState('active', parseAsBoolean.withDefault(false));
  const [sort]      = useQueryState('sort',   parseAsStringEnum(['asc', 'desc']).withDefault('asc'));
}
```

Browser back/forward and shareable URLs for free. Combine with regular `useState` for genuinely transient UI (e.g. hovered row).

```tsx
// ❌ useState for filters and sort.
// Share URL → recipient sees defaults. Refresh kills it. Back button doesn't restore.
```

---

## 8. Server State: TanStack Query

Server state has its own freshness lifecycle, can go stale, and is shared across components. `useEffect` + `useState` for fetching poorly reinvents caching, deduplication, retries, and race conditions.

```tsx
function UserList() {
  const { data: users, isLoading } = useQuery({
    queryKey: ['users'],
    queryFn:  fetchUsers,
  });

  if (isLoading) return <Spinner />;
  return users?.map((u) => <Row key={u.id} user={u} />);
}
```

Key knobs:

- **`queryKey`**: array of serialisable values that uniquely identifies the data. Same key = same cache entry = automatic dedup.
- **`staleTime`**: how long data is considered fresh before background refetch.
- **`retry`**: automatic retries with exponential backoff.
- **`useMutation`**: for writes; call `queryClient.invalidateQueries({ queryKey: ['users'] })` in `onSuccess`.

```tsx
// ❌ Manual fetching
useEffect(() => {
  setIsLoading(true); setError(null);
  fetchUsers()
    .then(setUsers)                  // race: deps may have changed mid-flight
    .catch((e) => setError(e.message))
    .finally(() => setIsLoading(false));
}, [query]);                         // no cache, no dedup, may setState on unmounted component
```

---

## 9. External Stores

When state has many cross-cutting consumers, complex transitions, or independent slices, React's built-in tools hit a ceiling. Context re-renders all consumers on any change; business logic gets tangled in components.

Two flavours:

- **Stores (centralised)**: one big state object, events update it, components subscribe with selectors. Best for complex interrelated state. (XState Store, Zustand, Redux Toolkit.)
- **Atoms (distributed)**: small independent pieces, composable. Best for independent UI bits. (Jotai, Recoil.)

```tsx
import { createStore } from '@xstate/store';
import { useSelector } from '@xstate/store/react';

const cartStore = createStore({
  context: { items: [] as Item[] },
  on: {
    add:    (ctx, e: { item: Item }) => ({ ...ctx, items: [...ctx.items, e.item] }),
    remove: (ctx, e: { sku: string }) => ({ ...ctx, items: ctx.items.filter((i) => i.sku !== e.sku) }),
    clear:  (ctx) => ({ ...ctx, items: [] }),
  },
});

// ✅ Selective subscription: only re-renders when `items` changes
function CartIcon() {
  const items = useSelector(cartStore, (s) => s.context.items);
  return <Badge count={items.length} />;
}
```

```tsx
// ❌ One giant AppContext with user, cart, notifications, and orders.
// Every update to any field re-renders every consumer.
```

---

## 10. Normalisation

Nested data (`users[].posts[]`) forces O(n×m) traversals for any update and recreates the whole parent array on every leaf change. Flatten into separate keyed collections referenced by id. Updates become O(1).

```tsx
// ❌ Nested
interface User { id: string; name: string; posts: { id: string; text: string }[] }

// ✅ Flat with foreign keys
interface User { id: UserId; name: string }
interface Post { id: PostId; text: string; userId: UserId }

interface AppState { users: User[]; posts: Post[] }
```

Branded ids for compile-time safety:

```tsx
type Brand<B> = string & { __brand: B };
type UserId = Brand<'UserId'>;
type PostId = Brand<'PostId'>;

const id = crypto.randomUUID() as PostId; // can't accidentally pass a UserId
```

Updates become trivial:

```tsx
case 'DELETE_POST':
  return { ...state, posts: state.posts.filter((p) => p.id !== action.postId) };

case 'ADD_POST':
  return {
    ...state,
    posts: [...state.posts, { id: crypto.randomUUID() as PostId, text: action.text, userId: action.userId }],
  };
```

Consumers derive via filter: `<PostList posts={state.posts.filter((p) => p.userId === user.id)} />`.

### Event sourcing for undo/redo

Keep an `events: Action[]` log; replay from `initialState` to undo:

```tsx
case 'UNDO': {
  const events = state.events.slice(0, -1);
  const undone = events.reduce(reducer, initialState);
  return { ...undone, events, undos: [...state.undos, state.events.at(-1)!] };
}
```

```tsx
// ❌ Mapping over users to find one, then mapping over posts to find one,
// returning new copies of both. Every user gets re-created on a single leaf change.
```

---

## 11. Effects: Avoid Cascading `useEffect`s

Multiple effects that each set state another effect depends on form a "cascade". Logic flow jumps unpredictably; races emerge; debugging is a nightmare. Replace the chain with a `useReducer` that encodes *why* state changes, plus a single effect that reads the current status.

### Reducer drives transitions

```tsx
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'inputChanged': {
      const inputs = { ...state.inputs, ...action.inputs };
      return {
        ...state,
        inputs,
        status: isComplete(inputs) ? 'searching' : state.status,
      };
    }
    case 'searchSucceeded': return { ...state, status: 'idle', results: action.results };
    case 'searchFailed':    return { ...state, status: 'error', error: action.error };
  }
}
```

### One effect, branches on status

```tsx
useEffect(() => {
  if (state.status !== 'searching') return;
  let cancelled = false;
  (async () => {
    try {
      const results = await search(state.inputs);
      if (!cancelled) dispatch({ type: 'searchSucceeded', results });
    } catch (error) {
      if (!cancelled) dispatch({ type: 'searchFailed', error: String(error) });
    }
  })();
  return () => { cancelled = true; };
}, [state]);
```

Think in *events* ("user submitted"), not *reactions* ("when X changes…"). Effects are for synchronising with the outside world (APIs, subscriptions), never for syncing internal state.

```tsx
// ❌ Four effects, four boolean flags, race-prone:
useEffect(() => { if (a && b) setIsSearching(true); }, [a, b]);
useEffect(() => { if (isSearching) fetch().then((r) => { setResult(r); setIsSearching(false); }); }, [isSearching]);
useEffect(() => { if (result) setIsLoading2(true); }, [result]);
// ...
```

---

## 12. `useSyncExternalStore`: External Data Sources

External data sources (browser APIs, third-party stores, WebSockets) need to reflect in React without effect-and-state race conditions or SSR hydration mismatches. `useSyncExternalStore` is the official primitive: `subscribe(cb)`, `getSnapshot()`, and optional `getServerSnapshot()`.

```tsx
export class Store<T> {
  private value: T;
  private listeners = new Set<() => void>();
  constructor(initial: T) { this.value = initial; }

  subscribe = (cb: () => void) => {
    this.listeners.add(cb);
    return () => this.listeners.delete(cb);
  };

  getSnapshot = () => this.value;

  set = (next: T) => {
    this.value = next;
    this.listeners.forEach((cb) => cb());
  };
}
```

Snapshots must be **stable** between calls: same reference if nothing changed.

```tsx
const onlineStore = new Store(navigator.onLine);
window.addEventListener('online',  () => onlineStore.set(true));
window.addEventListener('offline', () => onlineStore.set(false));

function useOnline() {
  return useSyncExternalStore(
    onlineStore.subscribe,
    onlineStore.getSnapshot,
    () => true, // SSR snapshot
  );
}
```

```tsx
// ❌ Manual effect + listener pair: races, hydration mismatch
const [isOnline, setIsOnline] = useState(true);
useEffect(() => {
  const on  = () => setIsOnline(true);
  const off = () => setIsOnline(false);
  window.addEventListener('online', on); window.addEventListener('offline', off);
  setIsOnline(navigator.onLine); // races with the listeners
  return () => { /* manual cleanup */ };
}, []);
```

---

## 13. Testing: Pure Reducers, No DOM

When business logic lives in a pure reducer separated from the UI, you can test it without React, the DOM, or mocks. Tests run in milliseconds, cover transitions exhaustively, and don't break when a button gets restyled. The unit under test is `(state, action) => state`.

```tsx
// cart.ts: no React imports
export const initialCart: CartState = { items: [], status: 'idle' };

export function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add':    return { ...state, items: [...state.items, { sku: action.sku, qty: 1 }] };
    case 'remove': return { ...state, items: state.items.filter((i) => i.sku !== action.sku) };
    case 'clear':  return { ...state, items: [] };
  }
}
```

```ts
import { test, expect } from 'vitest';
import { initialCart, cartReducer } from './cart';

test('add appends an item', () => {
  const state = cartReducer(initialCart, { type: 'add', sku: 'abc' });
  expect(state.items).toHaveLength(1);
});

test('remove is a no-op for unknown sku', () => {
  const state = cartReducer(initialCart, { type: 'remove', sku: 'nope' });
  expect(state).toEqual(initialCart);
});

test('clear empties the cart', () => {
  const seeded = cartReducer(initialCart, { type: 'add', sku: 'abc' });
  expect(cartReducer(seeded, { type: 'clear' }).items).toEqual([]);
});
```

Cover:

- Happy-path transitions between every adjacent state.
- Derived calculations (totals, can-proceed predicates).
- Business rules and validators.
- Edge cases: null/empty inputs, invalid transitions.

```tsx
// ❌ Logic mixed inside the component, forcing every test to render, fireEvent,
// screen.getByText. Slow, brittle, breaks on any UI tweak.
```

---

## 14. Where State Lives: Quick Reference

| Kind of state                                      | Where it belongs                       |
| -------------------------------------------------- | -------------------------------------- |
| Can be derived from other state                    | Computed in render; nowhere            |
| Doesn't trigger UI changes (timer ids, scroll pos) | `useRef`                               |
| Local to one component                             | `useState`                             |
| Coordinated across a feature                       | `useReducer` + Context                 |
| Shareable, bookmarkable, refresh-survivable        | URL (nuqs)                             |
| Owned by a remote server                           | TanStack Query                         |
| Many cross-cutting consumers, complex transitions  | External store (XState Store, Zustand) |
| Independent atomic pieces                          | Atom library (Jotai)                   |
| External non-React data source                     | `useSyncExternalStore`                 |
| Form data the browser already tracks               | `FormData` + server actions            |

---

## 15. Cardinal Rules

1. **Make impossible states impossible**: discriminated unions over boolean flags.
2. **Derive, don't store**: if a value can be computed from other state, compute it.
3. **Store ids, not objects**: look up by id at render time.
4. **Events, not reactions**: dispatch what happened; let the reducer decide the transition.
5. **One effect per concern**: effects synchronise with the outside world, not internal state.
6. **Lift to context after the second relay prop.**
7. **Pure reducers, tested without the DOM.**
8. **URL for anything shareable.** TanStack Query for anything remote.
9. **Flatten before you nest**: keyed collections plus foreign keys.
10. **Validate at the edge** with Zod; trust your types within.

---

## Further reading

- [TanStack Query](https://tanstack.com/query)
- [XState Store](https://stately.ai/docs/xstate-store)
- [nuqs](https://nuqs.47ng.com)
- [Zod](https://zod.dev)
- [React docs: `useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore)
- [React docs: `useActionState`](https://react.dev/reference/react/useActionState)
