# React + TypeScript Cheatsheet

Distilled from Steve Kinney's [React with TypeScript course](https://stevekinney.com/courses/react-typescript). Patterns use ✅ for recommended and ❌ for anti-patterns.

---

## 1. Project Setup

### tsconfig.json (React 19 + Vite)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "jsx": "react-jsx",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src/**/*"]
}
```

Key flags worth knowing:

- `noUncheckedIndexedAccess` — `arr[i]` returns `T | undefined`. Forces you to handle gaps.
- `exactOptionalPropertyTypes` — `foo?: string` no longer accepts `undefined` explicitly.
- `jsx: "react-jsx"` — no more `import React`.
- `isolatedModules` — required for SWC/esbuild.

### Migrating JS → TS incrementally

Start permissive, then tighten. Order: `noImplicitAny` → `strictNullChecks` → `noImplicitReturns` → `strictFunctionTypes` → full `strict`. Use `@ts-check` and JSDoc to type `.js` files before renaming.

```javascript
// @ts-check
/** @param {string} name */
function greet(name) { return `Hi ${name}`; }
```

---

## 2. Component Props

### Basic pattern

```tsx
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
}

function Button({ children, onClick, variant = 'primary', disabled = false }: ButtonProps) {
  return <button onClick={onClick} disabled={disabled} className={variant}>{children}</button>;
}
```

`interface` vs `type`: either works, but `interface` extends more ergonomically with `extends`.

### Children

```tsx
// ✅ Just use ReactNode
interface CardProps { title: string; children: React.ReactNode; }

// ✅ Or the helper
import { PropsWithChildren } from 'react';
function Card({ title, children }: PropsWithChildren<{ title: string }>) { ... }

// ❌ FC implicitly added children in old React types — and the React 18 fix means it no longer does. Skip React.FC.
const Card: React.FC<{ title: string }> = ({ title, children }) => ... // children not typed
```

### Defaults — destructuring, not `defaultProps`

```tsx
// ✅ Modern
function Button({ variant = 'primary' }: ButtonProps) { ... }

// ❌ Deprecated in function components
Button.defaultProps = { variant: 'primary' };
```

### Optional vs required

> Make props required when the component genuinely can't function without them.

```tsx
// ❌ Everything optional — caller can pass nothing
interface BadProps { value?: string; onChange?: (v: string) => void; }

// ✅ Required where it matters
interface GoodProps { value: string; onChange: (v: string) => void; placeholder?: string; }
```

### Don't double up booleans — use discriminated unions

```tsx
// ❌ Impossible states are representable
interface BadState { isLoading: boolean; isError: boolean; isSuccess: boolean; data?: T; }

// ✅ Impossible states are unrepresentable
type State<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```

### Document with JSDoc (shows in IntelliSense)

```tsx
interface ChartProps {
  /** Data points to display @example [{ x: 0, y: 10 }] */
  data: Array<{ x: number; y: number }>;
  /** @default { width: 600, height: 400 } */
  size?: { width: number; height: number };
}
```

---

## 3. Mirroring DOM Props

Inherit every native attribute (`onClick`, `disabled`, `aria-*`, `data-*`) instead of redefining them.

```tsx
import { ComponentPropsWithoutRef } from 'react';

// ✅ Inherits all native <button> props
interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary';
}

function Button({ variant = 'primary', className, ...rest }: ButtonProps) {
  return <button className={`btn btn--${variant} ${className ?? ''}`} {...rest} />;
}
```

### Omit when you need to replace a native prop

```tsx
// ✅ Replace native onChange (which gives an event) with a value-only callback
interface InputProps extends Omit<ComponentPropsWithoutRef<'input'>, 'onChange'> {
  onChange: (value: string) => void;
}
```

`ComponentPropsWithRef<'button'>` if you need ref typing baked in. Avoid raw `JSX.IntrinsicElements['button']` — `ComponentPropsWithoutRef` is the recommended public API.

---

## 4. JSX Element Types — ReactNode vs ReactElement vs JSX.Element

```
ReactNode  ⊃  ReactElement  ⊃  JSX.Element
(most permissive)              (most restrictive)
```

| Type | Use for |
|---|---|
| `ReactNode` | `children` and anything you just render (strings, numbers, null, arrays, elements) |
| `ReactElement` | Props you'll `cloneElement` or inspect |
| `JSX.Element` | Avoid — it doesn't include `null`/`string`/`number` |

```tsx
// ✅ ReactNode for children
interface CardProps { children: React.ReactNode; }

// ❌ ReactElement is too narrow
interface BadCard { children: React.ReactElement; }
<BadCard>hello</BadCard>          // ❌ string not assignable
<BadCard>{null}</BadCard>         // ❌ null not assignable

// ✅ ReactElement for cloning
function Modal({ trigger }: { trigger: React.ReactElement }) {
  return cloneElement(trigger, { onClick: openModal });
}

// ✅ Return type that allows null
function Maybe({ show }: { show: boolean }): React.ReactNode {
  if (!show) return null;
  return <div>...</div>;
}
```

---

## 5. useState

```tsx
// ✅ Let inference do the work for primitives
const [count, setCount] = useState(0);             // number
const [name, setName] = useState('');              // string

// ❌ Redundant annotation
const [name, setName] = useState<string>('');

// ✅ Empty arrays/objects need a hint
const [items, setItems] = useState<string[]>([]);
const [user, setUser] = useState<User | null>(null);

// ❌ Inferred as never[]
const [items, setItems] = useState([]);

// ✅ Union literal type
type Filter = 'all' | 'active' | 'completed';
const [filter, setFilter] = useState<Filter>('all');

// ✅ Lazy init — runs once
const [todos, setTodos] = useState<Todo[]>(() => {
  return JSON.parse(localStorage.getItem('todos') ?? '[]');
});

// ❌ Runs every render
const [todos, setTodos] = useState(JSON.parse(localStorage.getItem('todos') ?? '[]'));

// ✅ Functional updates infer prev
setTodos(prev => [...prev, newTodo]);

// ❌ Mutation
todos.push(newTodo); setTodos(todos);
```

---

## 6. useReducer & Action Creators

```tsx
type CounterAction =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'set'; value: number };

function reducer(state: number, action: CounterAction): number {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
    case 'set':       return action.value;
    default:
      const _exhaustive: never = action; // ✅ Compile error if a case is missed
      return state;
  }
}

// ✅ Derive action type from creators
const actions = {
  increment: () => ({ type: 'increment' } as const),
  set: (value: number) => ({ type: 'set', value } as const),
};
type Action = ReturnType<typeof actions[keyof typeof actions]>;
type Dispatch = React.Dispatch<Action>;
```

---

## 7. Discriminated Unions for State

Replace boolean soup with a tagged union, then exhaust with `switch`.

```tsx
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function UserView({ state }: { state: FetchState<User> }) {
  switch (state.status) {
    case 'idle':    return <p>Ready</p>;
    case 'loading': return <Spinner />;
    case 'success': return <h1>{state.data.name}</h1>;    // ✅ narrowed
    case 'error':   return <p>{state.error.message}</p>;  // ✅ narrowed
    default:
      const _: never = state; throw new Error('unhandled');
  }
}
```

---

## 8. Events & Forms

```tsx
import { ChangeEvent, FormEvent, MouseEvent, KeyboardEvent, FocusEvent } from 'react';

const onChange = (e: ChangeEvent<HTMLInputElement>) => setValue(e.currentTarget.value);
const onClick = (e: MouseEvent<HTMLButtonElement>) => console.log(e.clientX);
const onSubmit = (e: FormEvent<HTMLFormElement>) => { e.preventDefault(); };
const onKey = (e: KeyboardEvent<HTMLInputElement>) => { if (e.key === 'Enter') ... };
```

`currentTarget` is typed to the element the handler is on. `target` is the element that triggered (could be a child). Prefer `currentTarget` when you need typed access to the element's properties.

### Generic change handler

```tsx
const handleChange = <T extends HTMLInputElement | HTMLTextAreaElement>(e: ChangeEvent<T>) => {
  const { name, value } = e.target;
  setForm(prev => ({ ...prev, [name]: value }));
};
```

### Number inputs — strings in, numbers out

```tsx
// ❌ value is a string
<input type="number" value={age} onChange={e => setAge(e.target.value)} />

// ✅ Convert explicitly
const onChange = (e: ChangeEvent<HTMLInputElement>) => {
  const v = e.target.value;
  if (v === '') return setAge(0);
  const n = Number(v);
  if (!Number.isNaN(n)) setAge(n);
};
```

> HTML inputs always return strings. Don't fight it, convert it.

---

## 9. Refs & useImperativeHandle

```tsx
// ✅ DOM ref
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => { inputRef.current?.focus(); }, []);

// ❌ Untyped — `current` is any
const inputRef = useRef(null);

// ✅ Mutable value across renders (no re-render on change)
const timerRef = useRef<number | undefined>(undefined);

// ✅ Callback ref for measure-on-mount
const measuredRef = useCallback((el: HTMLDivElement | null) => {
  if (el) setSize(el.getBoundingClientRect());
}, []);
```

> Refs aren't available during the first render — they're set during the commit phase. Always null-check.

### useImperativeHandle: expose a curated API

```tsx
interface VideoHandle {
  play: () => void;
  pause: () => void;
  seek: (t: number) => void;
}

const Video = forwardRef<VideoHandle, { src: string }>(({ src }, ref) => {
  const el = useRef<HTMLVideoElement>(null);
  useImperativeHandle(ref, () => ({
    play:  () => el.current?.play(),
    pause: () => el.current?.pause(),
    seek:  (t) => { if (el.current) el.current.currentTime = t; },
  }), []);
  return <video ref={el} src={src} />;
});
```

---

## 10. forwardRef, memo, displayName

```tsx
// ✅ Correct order: memo wraps forwardRef
const Button = memo(forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', ...rest }, ref) => (
    <button ref={ref} className={`btn--${variant}`} {...rest} />
  )
));
Button.displayName = 'Button';

// ❌ Reversed — loses ref forwarding
const Bad = forwardRef(memo<ButtonProps>(props => <button {...props} />));
```

Always set `displayName` on `forwardRef`/`memo` components — DevTools shows them as `ForwardRef` otherwise.

> React 19 makes `ref` a regular prop, so `forwardRef` is no longer necessary for new components — but it remains the standard for libraries supporting React 18.

---

## 11. Polymorphic Components (`as` prop)

```tsx
import { ElementType, ComponentPropsWithoutRef } from 'react';

type PolyProps<T extends ElementType, P = {}> = P & { as?: T } & ComponentPropsWithoutRef<T>;

function Text<T extends ElementType = 'span'>({ as, ...rest }: PolyProps<T, { size?: 'sm' | 'lg' }>) {
  const Component = as ?? 'span';
  return <Component {...rest} />;
}

<Text>plain</Text>                          // span
<Text as="a" href="/x">link</Text>          // ✅ href is required for <a>
<Text as="button" onClick={fn}>btn</Text>   // ✅ onClick is typed for button
<Text as="a" />                             // ❌ href required by <a>
```

---

## 12. Compound Components & Slots

```tsx
const TabsContext = createContext<{ active: string; setActive: (id: string) => void } | null>(null);

function Tabs({ children, defaultId }: { children: ReactNode; defaultId: string }) {
  const [active, setActive] = useState(defaultId);
  return <TabsContext.Provider value={{ active, setActive }}>{children}</TabsContext.Provider>;
}

function TabList({ children }: { children: ReactNode }) { return <div role="tablist">{children}</div>; }
function Tab({ id, children }: { id: string; children: ReactNode }) { ... }
function Panel({ id, children }: { id: string; children: ReactNode }) { ... }

// ✅ Attach as static properties for ergonomics
Tabs.List = TabList;
Tabs.Tab  = Tab;
Tabs.Panel = Panel;

// Usage
<Tabs defaultId="a"><Tabs.List><Tabs.Tab id="a">A</Tabs.Tab></Tabs.List><Tabs.Panel id="a">...</Tabs.Panel></Tabs>
```

---

## 13. Context — Safer Pattern

```tsx
// ❌ Caller has to deal with undefined every time
const Ctx = createContext<UserCtx | undefined>(undefined);
const ctx = useContext(Ctx); // UserCtx | undefined

// ✅ Helper that throws if used outside provider
function createSafeContext<T>(name: string) {
  const Context = createContext<T | null>(null);
  Context.displayName = name;
  function useCtx(): T {
    const v = useContext(Context);
    if (v === null) throw new Error(`use${name} must be used inside <${name}Provider>`);
    return v;
  }
  return [Context.Provider, useCtx] as const;
}

const [UserProvider, useUser] = createSafeContext<UserCtx>('User');
```

### Split state + dispatch into two contexts

Components that only need `dispatch` won't re-render when state changes.

```tsx
const StateCtx    = createContext<State | null>(null);
const DispatchCtx = createContext<Dispatch<Action> | null>(null);
```

### Memoize provider value

```tsx
// ✅ Stable identity
const value = useMemo(() => ({ user, login, logout }), [user]);
return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
```

---

## 14. Custom Hooks

```tsx
// ✅ Tuple return for setter-style APIs
function useToggle(initial = false): [boolean, () => void] {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(s => !s), []);
  return [on, toggle];
}

// ✅ Object return when there are >2 things
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  return {
    count,
    increment: useCallback(() => setCount(c => c + 1), []),
    decrement: useCallback(() => setCount(c => c - 1), []),
    reset:     useCallback(() => setCount(initial), [initial]),
  };
}

// ✅ Generic fetch hook
function useFetch<T>(url: string) {
  const [state, setState] = useState<FetchState<T>>({ status: 'idle' });
  // ...
  return state;
}
```

---

## 15. Type Narrowing

```tsx
// typeof
if (typeof x === 'string') x.toUpperCase();

// instanceof
if (err instanceof ValidationError) console.log(err.field);

// 'in' operator
if ('href' in props) return <a href={props.href}>...</a>;

// User-defined type guard
function isUser(v: unknown): v is User {
  return typeof v === 'object' && v !== null && 'id' in v && 'name' in v;
}

// Non-null filter
function isNotNull<T>(v: T | null): v is T { return v !== null; }
const users = arr.filter(isNotNull); // ✅ User[]

// Assertion function
function assertDefined<T>(v: T | null | undefined): asserts v is T {
  if (v == null) throw new Error('expected value');
}
```

---

## 16. `unknown` vs `any`

```tsx
// ❌ any disables TypeScript
async function fetchUser(): Promise<any> { return (await fetch('/u')).json(); }
const u = await fetchUser();
u.namee.toUpperCase(); // No error, runtime explodes

// ✅ unknown forces validation
async function fetchUser(): Promise<unknown> { return (await fetch('/u')).json(); }
const data = await fetchUser();
if (isUser(data)) data.name.toUpperCase(); // narrowed

// ✅ catch
try { ... } catch (e: unknown) {
  if (e instanceof Error) console.log(e.message);
}
```

> Every `any` is a runtime error waiting to happen. Every `unknown` is a safety checkpoint.

---

## 17. Utility Types

```tsx
// Partial — for updates
const update = (patch: Partial<FormState>) => setState(s => ({ ...s, ...patch }));

// Pick — narrow a type for a component
type ArticleCard = Pick<Article, 'title' | 'excerpt' | 'author'>;

// Omit — strip internals or replace props
type PublicUser = Omit<User, 'password' | 'internalId'>;

// Required — flip optional fields back
type StrictConfig = Required<Config>;

// Readonly
const config: Readonly<Config> = { url: '...' };

// Record — keyed maps
const fields: Record<string, FormField> = {};

// Exclude / Extract — filter unions
type NonError = Exclude<FetchState<T>['status'], 'error'>;

// ReturnType / Parameters
type FetcherReturn = ReturnType<typeof fetchUser>;
type FetcherArgs   = Parameters<typeof fetchUser>;

// NonNullable
type Defined = NonNullable<string | null | undefined>; // string

// Combined
type UserCreate = Omit<User, 'id' | 'createdAt'>;
type UserUpdate = Partial<Pick<User, 'name' | 'email'>>;
```

---

## 18. Template Literal Types

```tsx
type Size = 'sm' | 'md' | 'lg';
type Color = 'primary' | 'danger';
type Variant = `${Size}-${Color}`; // 'sm-primary' | 'sm-danger' | ...

type EventName = `on${Capitalize<'click' | 'focus' | 'blur'>}`;
// 'onClick' | 'onFocus' | 'onBlur'

type CSSValue = `${number}${'px' | 'em' | 'rem' | '%'}`;
const w: CSSValue = '100px'; // ✅
const x: CSSValue = '100';   // ❌ missing unit
```

---

## 19. `as const` and `satisfies`

```tsx
// ✅ Narrow to literal types
const SIZES = ['sm', 'md', 'lg'] as const;
type Size = typeof SIZES[number]; // 'sm' | 'md' | 'lg'

// ❌ Widened to string[]
const SIZES = ['sm', 'md', 'lg'];

// ✅ satisfies — verify constraint, keep literal inference
const routes = {
  home: '/',
  user: '/users/:id',
} satisfies Record<string, `/${string}`>;
routes.home; // typed as the literal '/' — methods like .startsWith work fine

// ❌ Annotation widens
const routes: Record<string, string> = { home: '/' };
routes.home; // just `string`
```

---

## 20. Conditional & Mapped Types

```tsx
// Conditional
type Awaited<T> = T extends Promise<infer U> ? U : T;
type UserData = Awaited<ReturnType<typeof fetchUser>>;

// Mapped — turn each prop into a form field
type FormState<T> = { [K in keyof T]: { value: T[K]; touched: boolean; error?: string } };

// Deep readonly
type DeepReadonly<T> = { readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K] };
```

---

## 21. Structural Typing & Branded Types

```tsx
// Structural — extra properties are fine on variables
interface Config { url: string; timeout: number; }
const settings = { url: '/a', timeout: 5, extra: 1 };
const c: Config = settings; // ✅ structural match

// But object literals get excess-property checks
const c: Config = { url: '/a', timeout: 5, extra: 1 }; // ❌

// Branded types — prevent mixing string ids
type UserId = string & { readonly __brand: 'UserId' };
type PostId = string & { readonly __brand: 'PostId' };

const u = '123' as UserId;
const p = '456' as PostId;
function getUser(id: UserId) { ... }
getUser(u); // ✅
getUser(p); // ❌ branded mismatch
```

---

## 22. Conditional / Mutually Exclusive Props

```tsx
// ✅ XOR with `never`
type Controlled   = { open: boolean; onOpenChange: (v: boolean) => void; defaultOpen?: never };
type Uncontrolled = { open?: never; onOpenChange?: never; defaultOpen?: boolean };
type DialogProps  = { children: ReactNode } & (Controlled | Uncontrolled);

// Variant-based props
type ButtonProps =
  | { variant: 'button'; onClick: () => void }
  | { variant: 'link';   href: string };

// Accessibility constraint — icon-only buttons must have aria-label
type IconButton =
  | { icon: string; 'aria-label': string; children?: never }
  | { icon?: never; children: ReactNode };
```

---

## 23. Generic Components

```tsx
interface SelectProps<T> {
  options: T[];
  value?: T;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string | number;
}

function Select<T>({ options, getLabel, getValue, onChange }: SelectProps<T>) {
  return (
    <select onChange={e => {
      const selected = options.find(o => String(getValue(o)) === e.target.value);
      if (selected) onChange(selected);
    }}>
      {options.map(o => <option key={getValue(o)} value={getValue(o)}>{getLabel(o)}</option>)}
    </select>
  );
}
```

For generic `forwardRef`, you typically need an explicit cast:

```tsx
const Select = forwardRef(<T,>(props: SelectProps<T>, ref: Ref<HTMLSelectElement>) => ...)
  as <T>(p: SelectProps<T> & { ref?: Ref<HTMLSelectElement> }) => ReactElement;
```

---

## 24. Async Data, `use()` and Runtime Validation

### React 19 `use()` hook

```tsx
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // ✅ Suspends until resolved
  return <h1>{user.name}</h1>;
}

function App() {
  // ✅ Memoize so the same Promise reference is used across renders
  const promise = useMemo(() => fetchUser(id), [id]);
  return (
    <ErrorBoundary fallback={<Err />}>
      <Suspense fallback={<Spinner />}>
        <UserProfile userPromise={promise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Validate at the boundary with Zod

```tsx
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string(),
  email: z.string().email(),
});
type User = z.infer<typeof UserSchema>; // ✅ One source of truth

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(res.statusText);
  return UserSchema.parse(await res.json()); // throws on shape mismatch
}

// ✅ Non-throwing variant
const result = UserSchema.safeParse(data);
if (!result.success) console.error(result.error.flatten());
```

### Race-safe useEffect fetch

```tsx
useEffect(() => {
  const ctrl = new AbortController();
  fetch(url, { signal: ctrl.signal })
    .then(r => r.json())
    .then(setData)
    .catch(e => { if (e.name !== 'AbortError') setError(e); });
  return () => ctrl.abort();
}, [url]);
```

---

## 25. Error Boundaries

```tsx
interface State { hasError: boolean; error?: Error; }
interface Props {
  children: ReactNode;
  fallback: (e: Error, reset: () => void) => ReactNode;
  onError?: (e: Error, info: ErrorInfo) => void;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };
  static getDerivedStateFromError(error: Error): State { return { hasError: true, error }; }
  componentDidCatch(e: Error, info: ErrorInfo) { this.props.onError?.(e, info); }
  reset = () => this.setState({ hasError: false, error: undefined });
  render() {
    if (this.state.hasError && this.state.error)
      return this.props.fallback(this.state.error, this.reset);
    return this.props.children;
  }
}

// Combine
<ErrorBoundary fallback={(e, retry) => <Err err={e} onRetry={retry} />}>
  <Suspense fallback={<Spinner />}>
    <Data />
  </Suspense>
</ErrorBoundary>
```

---

## 26. Result / Either Pattern

```tsx
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

async function getUser(id: string): Promise<Result<User, 'NotFound' | 'Network'>> {
  try {
    const res = await fetch(`/users/${id}`);
    if (!res.ok) return { ok: false, error: 'NotFound' };
    return { ok: true, value: await res.json() };
  } catch { return { ok: false, error: 'Network' }; }
}

const r = await getUser('1');
if (r.ok) console.log(r.value.name); else console.log(r.error);
```

Or use `neverthrow` if you want `.map`/`.andThen` chains.

---

## 27. React Query

```tsx
import { useQuery, UseQueryResult } from '@tanstack/react-query';

function useUser(id: string): UseQueryResult<User, Error> {
  return useQuery<User, Error>({
    queryKey: ['user', id] as const,  // ✅ as const → literal type
    queryFn: () => fetchUser(id),
    staleTime: 60_000,
  });
}

// ✅ Type-narrow on retry
retry: (count, error) => error instanceof TRPCError && error.data?.httpStatus < 500 ? false : count < 3,
```

---

## 28. React Server Components & Actions

### Server components are async

```tsx
// ✅ Server component
export default async function ProductList() {
  const products = await db.products.findAll();
  return <ul>{products.map(p => <Product key={p.id} product={p} />)}</ul>;
}

// ✅ Client island
'use client';
export function AddToCart({ id }: { id: string }) {
  const [loading, setLoading] = useState(false);
  // ...
}
```

### Serializable props only across the boundary

```tsx
// ❌ Date and Function don't survive serialization
interface ServerOut { createdAt: Date; onClick: () => void; }

// ✅ Strings and primitives only
interface ClientSafeProduct { id: string; name: string; createdAt: string; }
```

### Server actions with `useActionState`

```tsx
'use client';
const [state, formAction, isPending] = useActionState(
  async (prev: ActionResult, formData: FormData): Promise<ActionResult> => {
    const parsed = PostSchema.safeParse(Object.fromEntries(formData));
    if (!parsed.success) return { success: false, fieldErrors: parsed.error.flatten().fieldErrors };
    return createPost(parsed.data);
  },
  { success: false },
);
```

### `useOptimistic`

```tsx
const [optimistic, addOptimistic] = useOptimistic(
  posts,
  (state: Post[], pending: Post) => [...state, pending],
);
```

---

## 29. Performance

```tsx
// ❌ Inline object — new identity every render
<Component config={{ theme: 'dark' }} />

// ✅ Hoisted constant
const CONFIG = { theme: 'dark' } as const;
<Component config={CONFIG} />

// ✅ Stable callback
const onSelect = useCallback((u: User) => setSelected(u.id), []);

// ✅ Memoized expensive computation
const total = useMemo(() => items.reduce((s, i) => s + i.value, 0), [items]);
```

> Measure first. memo/useMemo/useCallback add overhead — only reach for them on hot paths.

---

## 30. Accessibility Typing

React types `aria-*` permissively. Force important ones at compile time:

```tsx
// ✅ Buttons must have a label (visible or aria)
type LabelledButton =
  | { children: ReactNode }
  | { 'aria-label': string }
  | { 'aria-labelledby': string };

// ✅ Disable accidental link+button mixing
type LinkOrButton =
  | { href: string; onClick?: never }
  | { href?: never; onClick: () => void };
```

---

## 31. HOCs vs Custom Hooks vs Render Props

Prefer custom hooks. Use HOCs only when wrapping component behavior; use render props when consumers need to control rendering.

```tsx
// HOC — preserve generics, omit injected props
import { ComponentType } from 'react';

function withLoading<P extends object>(Wrapped: ComponentType<P>) {
  return function Wrapper(props: P & { isLoading?: boolean }) {
    const { isLoading, ...rest } = props;
    if (isLoading) return <Spinner />;
    return <Wrapped {...(rest as P)} />;
  };
}

// Render props with generics
interface FetchProps<T> {
  url: string;
  children: (state: FetchState<T>) => ReactNode;
}
function Fetch<T>({ url, children }: FetchProps<T>) { ... }
```

---

## 32. Routing & Env Vars

### Validated route params

```tsx
const paramsSchema = z.object({
  userId: z.string().transform(Number).pipe(z.number().positive()),
});

function useTypedParams<T>(schema: z.ZodSchema<T>): T {
  const params = useParams();
  return schema.parse(params);
}
```

### Env vars (Vite)

```tsx
// src/env.ts
const get = (k: string, fallback?: string) => {
  const v = import.meta.env[`VITE_${k}`] ?? fallback;
  if (v == null) throw new Error(`Missing VITE_${k}`);
  return v;
};

export const env = {
  apiUrl: get('API_URL'),
  enableAnalytics: get('ENABLE_ANALYTICS', 'false') === 'true',
} as const;
```

`vite-env.d.ts`:

```ts
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_ENABLE_ANALYTICS?: string;
}
interface ImportMeta { readonly env: ImportMetaEnv; }
```

---

## 33. Testing

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';

// ✅ Type props via the component's prop interface
const props: UserCardProps = { user: mockUser, onEdit: vi.fn() };
render(<UserCard {...props} />);

// ✅ getByRole returns HTMLElement; queryByX may be null
const btn = screen.getByRole('button');
const maybe = screen.queryByText('Optional');

// ✅ Cast when you need the specific element type
const input = screen.getByRole('textbox') as HTMLInputElement;

// ✅ Typed mocks
vi.mock('../api', () => ({ fetchUser: vi.fn() }));
const mocked = vi.mocked(fetchUser);
mocked.mockResolvedValue(mockUser);

// ✅ Mock factory keeps test data typed
const makeUser = (over: Partial<User> = {}): User => ({ id: '1', name: 'X', email: 'x@y', ...over });

// ✅ Hook tests
const { result } = renderHook(() => useCounter());
act(() => result.current.increment());
expect(result.current.count).toBe(1);
```

### Type-level testing

```tsx
import { expectTypeOf } from 'vitest';

expectTypeOf<ReturnType<typeof useCounter>>().toMatchTypeOf<{ count: number }>();

// Guard against accepting wrong props
// @ts-expect-error — variant must be a known value
<Button variant="bogus" />
```

---

## 34. Common Errors → Quick Fixes

| Error | Fix |
|---|---|
| `Property 'children' does not exist` | Add `children: ReactNode` or use `PropsWithChildren` |
| `Object is possibly 'null'` | Optional chain (`?.`) or guard before access |
| `Type 'X' is not assignable to 'MouseEventHandler'` | Use `React.MouseEvent<HTMLButtonElement>` |
| `Type 'T' does not satisfy the constraint` | Add generic constraint: `function f<T extends { id: string }>(...)` |
| `JSX element type does not have any construct signatures` | Import the actual component, don't store path as string |
| `Type 'string' is not assignable to type '"a" \| "b"'` | Use `as const` on the literal, or annotate the variable |
| `Argument of type 'undefined' is not assignable` (under exactOptionalPropertyTypes) | Don't pass `undefined` explicitly; omit the prop instead |

---

## 35. Cardinal Rules

1. **Make impossible states impossible** — discriminated unions over boolean flags.
2. **Be explicit at boundaries, implicit within implementations** — type props and exports; let inference handle locals.
3. **Validate at the edge** — every external `unknown` (fetch, localStorage, env) gets parsed with Zod or a type guard.
4. **Use `unknown`, never `any`.**
5. **Prefer `ComponentPropsWithoutRef<'tag'>` to hand-rolled prop lists.**
6. **`ReactNode` for children. `ReactElement` only when you'll inspect/clone.**
7. **Exhaustive switches end in `const _: never = action;`.**
8. **Memoize provider values; split state/dispatch contexts to limit re-renders.**
9. **`as const` for literal narrowing; `satisfies` to validate without widening.**
10. **PropTypes is dead — TypeScript replaces it entirely.**
