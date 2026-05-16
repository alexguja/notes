# State Management Cheatsheet

## Cross-cutting principles

A few ideas recur through every exercise — internalise these and the rest follows:

1. **State is a snapshot derived from events.** Events capture intent and survive as history; state is just a projection of that history.
2. **Make impossible states impossible.** Discriminated unions and explicit state machines beat boolean soup (`isLoading && isError && isSuccess`).
3. **Pure functions for app logic.** Reducers, selectors, validators — testable, replayable, framework-agnostic.
4. **Pick the right tool for the state's lifetime and scope.** Render-time derivation → ref → `useState` → URL → server-state library → external store. Reach for the smallest tool that fits.
5. **One `useEffect`, not many.** Effects synchronise with the outside world; they don't orchestrate internal transitions.

---

## 1. Anti-patterns: what NOT to put in `useState`

Most state bugs are self-inflicted. The fix is brutal minimalism: store the smallest set of values that can't be derived. Everything else is computed at render time, kept in a ref, or looked up by id.

### Derived state — calculate, don't store

```tsx
// ❌ Two sources of truth, sync via effect
const [tripItems, setTripItems] = useState<Item[]>([]);
const [totalCost, setTotalCost] = useState(0);
useEffect(() => {
  setTotalCost(tripItems.reduce((s, i) => s + i.cost, 0));
}, [tripItems]);

// ✅ Derive in render
const totalCost = tripItems.reduce((s, i) => s + i.cost, 0);
```

Only reach for `useMemo` if the calculation is genuinely expensive. Most aren't.

### Refs for values the UI doesn't render

`useState` triggers a re-render. `useRef` doesn't. Use a ref for timer ids, scroll positions, previous values, analytics counters — anything that survives between renders but isn't displayed.

```tsx
function BookingTimer() {
  const [timeLeft, setTimeLeft] = useState(300);
  const timerIdRef = useRef<NodeJS.Timeout | null>(null);

  const startTimer = () => {
    if (timerIdRef.current) clearInterval(timerIdRef.current);
    const id = setInterval(() => {
      setTimeLeft((prev) => {
        if (prev <= 1) {
          clearInterval(id);
          timerIdRef.current = null;
          return 0;
        }
        return prev - 1;
      });
    }, 1000);
    timerIdRef.current = id;
  };

  useEffect(() => () => {
    if (timerIdRef.current) clearInterval(timerIdRef.current);
  }, []);
}
```

### Store ids, derive objects

```tsx
const [hotels] = useState<Hotel[]>([...]);
const [selectedHotelId, setSelectedHotelId] = useState<string | null>(null);

// ✅ Look up by id at render time
const selectedHotel = hotels.find((h) => h.id === selectedHotelId);

// ❌ Two copies of the same object that can drift
const [selectedHotel, setSelectedHotel] = useState<Hotel | null>(null);
```

### Don't duplicate props/context into state

If a value already lives in props or context, render against it directly. Copying it into `useState` makes it stale the moment the source changes.

---

## 2. Modelling before coding — diagrams as docs

Before writing any state code, write a `flows.md` in version control. Plain text beats fancy tools — it lives in git, requires no plugins, and forces clarity. Three diagrams cover most cases:

**Entity-Relationship (ERD)** — what exists and how it relates:

```
- Destination { id }
- Home        { id, destinationId }
- User        { id, name }
```

**Sequence diagram** — who calls whom, in order:

```
UI                -> SearchService : search(query, dates, guests)
SearchService     -> UI            : results[]
UI                -> UI            : selectHome(id)
UI                -> BookingAPI    : book(homeId, dates)
BookingAPI        -> PaymentSvc    : charge(amount)
```

**State diagram** — every state, what's stored in it, what events leave it:

```
idle
  shows: search form
  stores: { destination?, dates?, guests? }
  on submit -> searching

searching
  shows: spinner
  effect: fetch results
  on success -> results
  on failure -> error

results
  shows: list of homes
  on selectHome -> details
```

Modelling separates *incidental* complexity (irreducible domain logic) from *accidental* complexity (self-inflicted implementation choices). Done in 10 minutes; saves hours.

---

## 3. Finite states — discriminated unions over booleans

Replace many `useState` calls and boolean flags with one combined state object plus a discriminated union for status. Boolean combinations silently allow impossible states; a single `status` field with a string literal makes those states unreachable.

```tsx
// ❌ Boolean soup — what does isLoading=true, error="oops", data=[...] mean?
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<string | null>(null);
const [data, setData] = useState<Flight[] | null>(null);

// ✅ One discriminated union — each variant carries exactly what it needs
type FlightState = FlightData & (
  | { status: 'idle' }
  | { status: 'submitting'; selectedFlightId: null }
  | { status: 'error' }
  | { status: 'success'; flights: FlightOption[] }
);

const [flightState, setFlightState] = useState<FlightState>({
  status: 'idle',
  destination: '',
  departure: '',
  arrival: '',
  passengers: 1,
  isRoundtrip: false,
  selectedFlightId: null,
});

// `flights` is only accessible when status === 'success' — TS enforces it
const selectedFlight =
  flightState.status === 'success' && flightState.selectedFlightId
    ? flightState.flights.find((f) => f.id === flightState.selectedFlightId)
    : null;

const totalPrice = selectedFlight ? selectedFlight.price * flightState.passengers : 0;
```

Submit becomes a single atomic transition rather than three separate `set*` calls:

```tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  setFlightState((prev) => ({ ...prev, status: 'submitting', selectedFlightId: null }));
  try {
    const flights = await getFlightOptions(flightState);
    setFlightState((prev) => ({ ...prev, status: 'success', flights }));
  } catch {
    setFlightState((prev) => ({ ...prev, status: 'error' }));
  }
};
```

---

## 4. `useReducer` + Context — when state has structure

Once state has structure (events, transitions, multiple consumers), `useState` stops scaling. `useReducer` puts all transitions in one pure function (testable, deterministic, replayable). React Context plus a custom hook eliminates prop drilling. The reducer **is** the state machine.

### The action type

Actions describe *intent*, not assignments. Tag them with a discriminated `type` so the reducer narrows the payload per case.

```tsx
type BookingEvent =
  | { type: 'submit'; payload: SearchParams }
  | { type: 'results'; flightOptions: FlightOption[] }
  | { type: 'back' }
  | { type: 'error' };
```

### The reducer

```tsx
function bookingReducer(state: BookingState, event: BookingEvent): BookingState {
  switch (event.type) {
    case 'submit':
      return { ...state, status: 'submitting', searchParams: event.payload };
    case 'results':
      return { ...state, status: 'results', flightOptions: event.flightOptions };
    case 'back':
      return state.status === 'results' ? { ...state, status: 'idle' } : state;
    case 'error':
      return { ...state, status: 'error' };
    default:
      const _exhaustive: never = event; // compile error if a case is missed
      return state;
  }
}
```

### The provider + custom hook

```tsx
const BookingContext = createContext<{
  state: BookingState;
  dispatch: (e: BookingEvent) => void;
} | null>(null);

export function BookingProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(bookingReducer, initialBookingState);
  return (
    <BookingContext.Provider value={{ state, dispatch }}>
      {children}
    </BookingContext.Provider>
  );
}

export function useBooking() {
  const ctx = useContext(BookingContext);
  if (!ctx) throw new Error('useBooking must be used inside BookingProvider');
  return ctx;
}
```

### React 19 — `use()` replaces `useContext`

```tsx
function BookingContent() {
  const { state, dispatch } = use(BookingContext);
  // ...
}
```

> **Anti-pattern:** prop-drilling `state` and `setState` through `BookingPage → FlightForm → FlightFormFields`, each layer just relaying. If you're typing the same prop more than twice, lift it into context.

---

## 5. Forms — FormData, Server Actions, Zod

Controlled inputs with one `useState` per field is boilerplate that grows linearly. The platform already gives you `FormData` for free. Next.js server actions submit the form directly to a server function; `useActionState` handles pending and response state; Zod validates server-side. The whole thing degrades gracefully without JavaScript.

### The form

```tsx
const initialState: FormState = { status: 'idle', errors: {}, data: null };

export default function TravelFormPage() {
  const [state, submitAction, isPending] = useActionState(submitTravelData, initialState);

  if (state.status === 'success') return <SuccessCard data={state.data} />;

  return (
    <form action={submitAction} className="space-y-6">
      <Label htmlFor="firstName">First name *</Label>
      <Input
        id="firstName"
        name="firstName"
        defaultValue={state.submittedData?.firstName ?? ''}
        aria-invalid={state.errors?.firstName ? 'true' : 'false'}
        disabled={isPending}
      />
      {state.errors?.firstName && (
        <p className="text-sm text-red-600">{state.errors.firstName}</p>
      )}
      <Button type="submit" disabled={isPending}>
        {isPending ? 'Submitting…' : 'Submit'}
      </Button>
    </form>
  );
}
```

Note: `name="firstName"` and `defaultValue` (uncontrolled). `FormData` picks them up by name.

### The server action

```tsx
'use server';

const travelFormSchema = z.object({
  firstName: z.string().min(1),
  lastName:  z.string().min(1),
  email:     z.string().email(),
});

export async function submitTravelData(prev: FormState, formData: FormData): Promise<FormState> {
  const result = travelFormSchema.safeParse(Object.fromEntries(formData));
  if (!result.success) {
    return {
      status: 'error',
      errors: result.error.flatten().fieldErrors,
      submittedData: Object.fromEntries(formData) as Partial<TravelData>,
    };
  }
  await saveTravelData(result.data);
  return { status: 'success', data: result.data, errors: {} };
}
```

Zod doubles as runtime validation and the TS type — `type TravelData = z.infer<typeof travelFormSchema>`.

> **Anti-pattern:** one `useState` per field, one `onChange` per field, one piece of error state per field, plus your own `isSubmitting` flag. You're reinventing the form element.

---

## 6. URL state — `nuqs`

State that should be shareable, bookmarkable, or survive a refresh belongs in the URL, not `useState`. Search filters, pagination, tabs, form drafts — all URL state. `nuqs` makes it type-safe with parsers per query param.

```tsx
import { useQueryState, parseAsBoolean, parseAsInteger, parseAsStringEnum } from 'nuqs';

function BookingForm() {
  const [destination, setDestination] = useQueryState('destination');
  const [departure,   setDeparture]   = useQueryState('departure');
  const [arrival,     setArrival]     = useQueryState('arrival');
  const [passengers,  setPassengers]  = useQueryState('passengers', parseAsInteger.withDefault(1));
  const [isOneWay,    setIsOneWay]    = useQueryState('isOneWay',   parseAsBoolean.withDefault(false));
}

function SearchResults({ flightOptions }: { flightOptions: FlightOption[] }) {
  const [showDirectOnly] = useQueryState('directOnly', parseAsBoolean.withDefault(false));
  const [sortBy]    = useQueryState('sortBy',    parseAsStringEnum(['price', 'duration']).withDefault('price'));
  const [sortOrder] = useQueryState('sortOrder', parseAsStringEnum(['asc', 'desc']).withDefault('asc'));

  const filtered = flightOptions
    .filter((f) => !showDirectOnly || f.layovers.length === 0)
    .sort((a, b) => sortBy === 'price'
      ? (sortOrder === 'asc' ? a.price - b.price : b.price - a.price)
      : (sortOrder === 'asc' ? a.duration - b.duration : b.duration - a.duration));
}
```

You get browser back/forward and shareable URLs for free. Combine with regular `useState` for genuinely transient UI (e.g. hovered row).

> **Anti-pattern:** `useState` for filters and sort. User shares the URL → recipient sees defaults. Refresh kills the search. Back button doesn't restore.

---

## 7. Server state — TanStack Query

Server state is fundamentally different from client state. It's owned remotely, has its own freshness lifecycle, can go stale, and is shared across components. `useEffect` + `useState` for fetching poorly reinvents caching, deduplication, retries, and race conditions. TanStack Query owns this concern.

```tsx
function FlightSearchResults() {
  const { state, dispatch } = use(BookingContext)!;
  const { flightSearch, selectedFlight } = state;

  const { data: flights, isLoading } = useQuery({
    queryKey: ['flights', flightSearch],
    queryFn:  () => fetchFlights(flightSearch),
  });

  if (isLoading) return <Spinner />;

  return (
    <div className="space-y-4">
      {flights?.map((flight) => (
        <div
          key={flight.id}
          className={selectedFlight?.id === flight.id ? 'border-blue-500 bg-blue-50' : ''}
        >
          <h3>{flight.airline}</h3>
          <p>${flight.price}</p>
          <Button onClick={() => dispatch({ type: 'flightSelected', flight })}>Select</Button>
        </div>
      ))}
    </div>
  );
}
```

Key knobs:

- **`queryKey`** — array of serialisable values that uniquely identifies the data. Same key = same cache entry = automatic dedup.
- **`staleTime`** — how long data is considered fresh before background refetching.
- **`retry`** — automatic retries with exponential backoff.
- **`useMutation`** — for writes; call `queryClient.invalidateQueries({ queryKey: ['flights'] })` in `onSuccess` to refresh.

> **Anti-pattern:**
>
> ```tsx
> useEffect(() => {
>   setIsLoading(true); setError(null);
>   fetchFlights(flightSearch)
>     .then(setFlights)               // race: flightSearch may have changed mid-flight
>     .catch((e) => setError(e.message))
>     .finally(() => setIsLoading(false));
> }, [flightSearch]); // no cache, no dedup, may setState on unmounted component
> ```

---

## 8. External stores — XState Store, Zustand, Jotai

When state has many cross-cutting consumers, complex transitions, or independent slices, React's built-in tools hit a ceiling. Context re-renders all consumers on any change; prop drilling re-emerges; business logic gets tangled in components. External stores offer a single source of truth, selective subscriptions, dev tools, and framework independence.

Two flavours:

- **Stores (centralised)** — one big state object, events update it, components subscribe with selectors. Best for complex interrelated state. (XState Store, Zustand, Redux Toolkit.)
- **Atoms (distributed)** — small independent pieces of state, composable. Best for independent UI bits. (Jotai, Recoil.)

### XState Store example

```tsx
import { createStore } from '@xstate/store';
import { useSelector } from '@xstate/store/react';

const bookingStore = createStore({
  context: initialState,
  on: {
    flightSearchUpdated: (context, event: Partial<FlightSearch>) => ({
      ...context,
      flightSearch: { ...context.flightSearch, ...event },
    }),
    searchFlights: (context) => ({ ...context, currentStep: Step.FlightResults }),
    flightSelected: (context, event: { flight: FlightOption }) => ({
      ...context,
      selectedFlight: event.flight,
      currentStep: Step.HotelSearch,
    }),
    back: (context) => {
      switch (context.currentStep) {
        case Step.FlightResults: return { ...context, currentStep: Step.FlightSearch };
        case Step.HotelResults:  return { ...context, currentStep: Step.HotelSearch };
        case Step.Review:        return { ...context, currentStep: Step.HotelResults };
        default: return context;
      }
    },
  },
});

// Selective subscription — only re-renders when `flightSearch` changes
function FlightBookingForm() {
  const flightSearch = useSelector(bookingStore, (state) => state.context.flightSearch);
  return (
    <form onSubmit={(e) => { e.preventDefault(); bookingStore.trigger.searchFlights(); }}>
      <Input
        value={flightSearch.destination}
        onChange={(e) => bookingStore.trigger.flightSearchUpdated({ destination: e.target.value })}
      />
    </form>
  );
}
```

> **Anti-pattern:** one giant `AppContext` with user, cart, notifications, and orders. Every update to any field re-renders every consumer — you've centralised everything into the slowest possible context value.

---

## 9. Normalisation — flatten nested data

Nested data (`destinations[].todos[]`) forces O(n×m) traversals for any update and recreates the whole parent array on every leaf change. Flatten into separate keyed collections of entities that reference each other by id. Updates become O(1); unrelated entities don't change reference.

### Shape

```tsx
// ❌ Nested
interface Destination {
  id: string;
  name: string;
  todos: { id: string; text: string }[];
}

// ✅ Flat with foreign keys
interface Destination { id: DestinationId; name: string }
interface TodoItem    { id: TodoId; text: string; destinationId: DestinationId }

interface ItineraryState {
  destinations: Destination[];
  todos: TodoItem[];
}
```

### Branded ids for compile-time safety

```tsx
type Brand<B> = string & { __brand: B };
type DestinationId = Brand<'DestinationId'>;
type TodoId        = Brand<'TodoId'>;

const id = crypto.randomUUID() as TodoId; // can't accidentally pass a DestinationId
```

### Updates become trivial

```tsx
case 'DELETE_TODO':
  return { ...state, todos: state.todos.filter((t) => t.id !== action.todoId) };

case 'ADD_TODO':
  return {
    ...state,
    todos: [...state.todos, {
      id: crypto.randomUUID() as TodoId,
      text: action.text,
      destinationId: action.destinationId,
    }],
  };
```

Consumers derive via filter:

```tsx
<TodoList todos={state.todos.filter((t) => t.destinationId === destination.id)} />
```

### Event sourcing for undo/redo

Keep an `events: Action[]` log. To undo, replay all events except the last one from `initialState`:

```tsx
if (action.type === 'UNDO') {
  let undoneState = initialState;
  const events: Action[] = [];
  const undos = [...state.undos];
  for (let i = 0; i < state.events.length; i++) {
    if (i === state.events.length - 1) undos.push(state.events[i]);
    else {
      undoneState = itineraryReducer(undoneState, state.events[i]);
      events.push(state.events[i]);
    }
  }
  return { ...undoneState, events, undos };
}
```

> **Anti-pattern:** mapping over `destinations` to find the right one, mapping over its `todos` to find the right one, returning new copies of both. Every destination gets re-created when a single deeply-nested todo changes.

---

## 10. Effects — avoid cascading `useEffect`s

Multiple `useEffect` hooks that each set state that another effect depends on form a "cascade" — logic flow jumps unpredictably between effects, race conditions emerge, and debugging is a nightmare. Replace the chain with a `useReducer` (which encodes *why* state changes) plus a single `useEffect` that reads the current status and fires the right side effect.

### The reducer drives transitions

```tsx
function tripSearchReducer(state: BookingState, action: Action): BookingState {
  switch (action.type) {
    case 'inputUpdated': {
      const inputs = { ...state.inputs, ...action.inputs };
      return {
        ...state,
        inputs,
        status:
          inputs.destination && inputs.startDate && inputs.endDate
            ? 'searchingFlights'   // transition is driven by data shape
            : state.status,
      };
    }
    case 'flightUpdated':
      return { ...state, status: 'searchingHotels', selectedFlight: action.flight };
    case 'hotelUpdated':
      return { ...state, status: 'idle', selectedHotel: action.hotel };
    case 'error':
      return { ...state, status: 'error', error: action.error };
  }
}
```

### One effect, branching on status

```tsx
export default function TripSearch() {
  const [state, dispatch] = useReducer(tripSearchReducer, initialState);

  useEffect(() => {
    if (state.status === 'searchingFlights') {
      (async () => {
        try {
          const flights = await fetchFlights();
          const best = flights.reduce((p, c) => (p.price < c.price ? p : c));
          dispatch({ type: 'flightUpdated', flight: best });
        } catch {
          dispatch({ type: 'error', error: 'Failed to search flights' });
        }
      })();
    }
    if (state.status === 'searchingHotels') {
      (async () => {
        try {
          const hotels = await fetchHotels();
          const best = hotels.reduce((p, c) => (p.rating > c.rating ? p : c));
          dispatch({ type: 'hotelUpdated', hotel: best });
        } catch {
          dispatch({ type: 'error', error: 'Failed to search hotels' });
        }
      })();
    }
  }, [state]);
}
```

> Think in *events* ("user selected a flight"), not *reactions* ("when selectedFlight changes…"). Effects are for synchronising with the outside world (APIs, subscriptions), never for syncing internal state.

> **Anti-pattern:**
>
> ```tsx
> useEffect(() => {
>   if (destination && startDate && endDate) setIsSearchingFlights(true);
> }, [destination, startDate, endDate]);
>
> useEffect(() => {
>   if (!isSearchingFlights) return;
>   fetchFlights().then((f) => { setSelectedFlight(f); setIsSearchingFlights(false); });
> }, [isSearchingFlights]);
>
> useEffect(() => {
>   if (selectedFlight) setIsSearchingHotels(true);
> }, [selectedFlight]);
>
> useEffect(() => {
>   if (!isSearchingHotels) return;
>   fetchHotels().then((h) => { setSelectedHotel(h); setIsSearchingHotels(false); });
> }, [isSearchingHotels]);
> ```
>
> Four effects, four boolean flags, impossible to follow, race-prone. Changing a date mid-flight produces stale results.

---

## 11. `useSyncExternalStore` — external data sources

External data sources (browser APIs, third-party stores, WebSockets) need to reflect in React without `useEffect` + `useState` race conditions or SSR hydration mismatches. `useSyncExternalStore` is the official primitive: a `subscribe(cb)` function, a `getSnapshot()` for the client, and an optional `getServerSnapshot()` for SSR.

### The store contract

```tsx
export class FlightStore {
  private flights: Flight[] = [/* … */];
  private listeners = new Set<() => void>();

  subscribe = (callback: () => void) => {
    this.listeners.add(callback);
    return () => { this.listeners.delete(callback); };
  };

  getSnapshot = () => this.flights;

  private notify = () => { this.listeners.forEach((cb) => cb()); };

  updateFlightStatus = (flightId: string, status: FlightStatus) => {
    const flight = this.flights.find((f) => f.id === flightId);
    if (!flight) return;
    flight.status = status;
    this.flights = [...this.flights]; // new ref so React sees the change
    this.notify();
  };
}
```

The snapshot must be **stable** between calls — same reference if nothing changed. Mutate-then-clone on update.

### Consuming

```tsx
const flightStore = new FlightStore();

function useFlights() {
  return useSyncExternalStore(
    flightStore.subscribe,
    flightStore.getSnapshot,
    flightStore.getSnapshot, // SSR snapshot
  );
}

export default function FlightStatusDashboard() {
  const flights = useFlights();
  const delayed = flights.filter((f): f is Flight & { delay: number } => f.delay !== undefined);
  const avgDelay = delayed.reduce((acc, f) => acc + f.delay, 0) / delayed.length;
  // …live-updating dashboard
}
```

The same pattern wraps browser APIs (`navigator.onLine`, window size, geolocation), real-time feeds, or any non-React data source.

> **Anti-pattern:**
>
> ```tsx
> const [isOnline, setIsOnline] = useState(true);
> useEffect(() => {
>   const onUp = () => setIsOnline(true);
>   const onDown = () => setIsOnline(false);
>   window.addEventListener('online',  onUp);
>   window.addEventListener('offline', onDown);
>   setIsOnline(navigator.onLine); // races with the listeners
>   return () => { /* manual cleanup */ };
> }, []);
> ```
>
> SSR renders `true`, client may be offline → hydration mismatch. Manual sync is race-prone.

---

## 12. Testing — pure reducers, no DOM

When business logic lives in a pure reducer separated from the UI, you can test it without React, without the DOM, and without mocks. Tests run in milliseconds, exhaustively cover state transitions, and don't break when you restyle a button. The unit under test is `(state, action) => state` — a function.

### Extract the reducer

```tsx
// bookingFlow.ts — no React imports
export const initialState: BookingState = {
  currentStep: Step.FlightSearch,
  flightSearch: { destination: '', departure: '', arrival: '', passengers: 1, isOneWay: false },
  selectedFlight: null,
  hotelSearch:   { checkIn: '', checkOut: '', guests: 1, roomType: 'standard' },
  selectedHotel: null,
};

export function bookingReducer(state: BookingState, action: BookingAction): BookingState {
  switch (action.type) {
    case 'flightSearchUpdated':
      return { ...state, flightSearch: { ...state.flightSearch, ...action.payload } };
    case 'searchFlights':
      return { ...state, currentStep: Step.FlightResults };
    case 'flightSelected':
      return { ...state, selectedFlight: action.payload.flight, currentStep: Step.HotelSearch };
    case 'hotelSelected':
      return { ...state, selectedHotel: action.payload.hotel, currentStep: Step.Review };
    case 'book':
      return { ...state, currentStep: Step.Confirmation };
    case 'back':
      switch (state.currentStep) {
        case Step.FlightResults: return { ...state, currentStep: Step.FlightSearch };
        case Step.HotelSearch:   return { ...state, currentStep: Step.FlightResults };
        case Step.HotelResults:  return { ...state, currentStep: Step.HotelSearch };
        case Step.Review:        return { ...state, currentStep: Step.HotelResults };
        default: return state;
      }
  }
}
```

### Test the reducer directly

```ts
import { test, expect } from 'vitest';
import { initialState, bookingReducer, Step } from './bookingFlow';

test('searchFlights transitions to FlightResults', () => {
  const state = bookingReducer(initialState, { type: 'searchFlights' });
  expect(state.currentStep).toBe(Step.FlightResults);
});

test('hotel dates sync with flight dates on submission', () => {
  const state = {
    ...initialState,
    flightSearch: { ...initialState.flightSearch, departure: '2026-06-01', arrival: '2026-06-10' },
  };
  const result = bookingReducer(state, { type: 'flightSearchSubmitted' });
  expect(result.hotelSearch.checkIn).toBe('2026-06-01');
  expect(result.hotelSearch.checkOut).toBe('2026-06-10');
});

test('cannot proceed to booking without flight selection', () => {
  const state = { ...initialState, selectedFlight: null, selectedHotel: mockHotel };
  expect(canProceedToBooking(state)).toBe(false);
});

test('invalid step transitions are ignored', () => {
  const state = { ...initialState, currentStep: Step.FlightSearch };
  const result = bookingReducer(state, { type: 'book' }); // can't book from search
  expect(result.currentStep).toBe(Step.FlightSearch);
});
```

What to cover:

- Happy-path transitions between every adjacent state.
- Derived calculations (`calculateTotalCost`, `canProceedToBooking`).
- Business rules and validation helpers.
- Edge cases — null/empty inputs, invalid step transitions.
- Data merging during transitions (hotel dates inheriting from flight dates).

> **Anti-pattern:** logic mixed inside the component, forcing every test to `render`, `fireEvent`, `screen.getByText`. Slow, brittle, breaks on any UI tweak.

---

## Quick reference — picking where state lives

| Kind of state                                      | Where it belongs                       |
| -------------------------------------------------- | -------------------------------------- |
| Can be derived from other state                    | Computed in render; nowhere            |
| Doesn't trigger UI changes (timer ids, scroll pos) | `useRef`                               |
| Local to one component                             | `useState`                             |
| Coordinated across a feature                       | `useReducer` + Context                 |
| Shareable / bookmarkable / refresh-survivable      | URL (nuqs)                             |
| Owned by a remote server                           | TanStack Query                         |
| Many cross-cutting consumers, complex transitions  | External store (XState Store, Zustand) |
| Independent atomic pieces                          | Atom library (Jotai)                   |
| External non-React data source                     | `useSyncExternalStore`                 |
| Form data the browser already tracks               | `FormData` + server actions            |

---

## Further reading

- [TanStack Query](https://tanstack.com/query)
- [XState Store](https://stately.ai/docs/xstate-store)
- [nuqs](https://nuqs.47ng.com)
- [Zod](https://zod.dev)
- [React docs: `useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore)
- [React docs: `useActionState`](https://react.dev/reference/react/useActionState)
