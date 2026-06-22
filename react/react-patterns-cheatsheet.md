# React Patterns Cheatsheet

Distilled from Nadia Makarevich's [_Advanced React_](https://www.advanced-react.com). Patterns use ✅ for recommended and ❌ for anti-patterns. The throughline is **performance via composition**, not memoization.

---

## 1. The Re-render Mental Model

1. **State update is the only source of re-renders.** Props changing does nothing on its own.
2. **Re-renders cascade _down_, never up or sideways.** When a component re-renders, React re-renders every component it renders inside, regardless of props.
3. **A re-render is just React calling your function again.** Cheap by default; the cost is in what that function does and how deep the subtree is.
4. **A component re-renders when its element object changes** (`Object.is` on the element before vs after). Same reference → React bails out.
5. **Reach for composition before memoization.** `React.memo` is the last resort, not the first.

```tsx
// A state update in App re-renders App AND everything below it,
// even components with no props at all.
const App = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsOpen(true)} />
      <VerySlowComponent /> {/* re-renders too, for no reason */}
    </>
  );
};
```

---

## 2. Move State Down

Isolate state (and the components that use it) into the smallest possible child, so expensive siblings sit outside the re-render zone.

```tsx
// ❌ isOpen lives in App, so the whole app re-renders on every toggle
const App = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <div>
      <Button onClick={() => setIsOpen(true)}>Open</Button>
      {isOpen && <ModalDialog onClose={() => setIsOpen(false)} />}
      <VerySlowComponent />
    </div>
  );
};

// ✅ state isolated; VerySlowComponent never re-renders on toggle
const ButtonWithModal = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsOpen(true)}>Open</Button>
      {isOpen && <ModalDialog onClose={() => setIsOpen(false)} />}
    </>
  );
};

const App = () => (
  <div>
    <ButtonWithModal />
    <VerySlowComponent />
  </div>
);
```

---

## 3. Components/Children as Props

If state must live in a parent (scroll position, mouse coords), pass the slow components _in_ as `children`. Elements passed as props are created by the **caller**, so the parent's re-render doesn't change their reference → they don't re-render.

```tsx
// ✅ position changes re-render the wrapper, but `children` is a stable
//    element created in App, so VerySlowComponent is untouched
const ScrollArea = ({ children }) => {
  const [position, setPosition] = useState(0);
  return (
    <div onScroll={(e) => setPosition(e.currentTarget.scrollTop)}>
      <MovingBlock position={position} />
      {children}
    </div>
  );
};

const App = () => (
  <ScrollArea>
    <VerySlowComponent />
  </ScrollArea>
);
```

`children` is just a prop — `<Parent><Child/></Parent>` is identical to `<Parent children={<Child/>} />`.

---

## 4. Elements as Props (Configuration)

When a component sprouts endless config props (`iconColor`, `iconSize`, `iconName`...), pass the element instead and hand control to the consumer.

```tsx
// ❌ const Button = ({ iconName, iconColor, iconSize, isIconAvatar, ... }) => ...

// ✅ consumer owns the icon entirely
const Button = ({ icon }: { icon: ReactElement }) => <button>Submit {icon}</button>;

<Button icon={<Error color="red" size="large" />} />;
```

- **Conditional rendering is free:** an element assigned to a variable only renders when the component holding it actually renders. `const footer = <Footer />` then `isOpen ? <Dialog footer={footer} /> : null` — `Footer` mounts only when the dialog opens.
- **Defaults via `cloneElement`** (fragile — keep it to trivial cases): `cloneElement(icon, { ...defaults, ...icon.props })`.

---

## 5. Render Props

Use when the parent needs to pass its **own state/props into** the element it receives — something plain elements-as-props can't do.

```tsx
const Button = ({ renderIcon }: { renderIcon: (p: IconProps) => ReactElement }) => {
  const [isHovered, setIsHovered] = useState(false);
  return (
    <button onMouseEnter={() => setIsHovered(true)}>
      Submit {renderIcon({ isHovered })}
    </button>
  );
};

<Button renderIcon={({ isHovered }) => <HomeIcon highlighted={isHovered} />} />;

// children-as-render-prop: share state without lifting it up
const MousePosition = ({ children }: { children: (p: Point) => ReactElement }) => {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>{children(pos)}</div>;
};

<MousePosition>{({ x }) => <span>{x}</span>}</MousePosition>;
```

> Hooks replaced render props for ~99% of stateful-logic sharing. Render props still earn their keep when the logic is tied to a specific DOM element.

---

## 6. Higher-Order Components

A function that takes a component and returns an enhanced one. Modern uses: injecting props, intercepting events/lifecycle. (Hooks cover most cases now, but HOCs still shine for cross-cutting wrappers.)

```tsx
const withLogOnClick = (Component) => (props) => {
  const onClick = () => {
    console.log("clicked");
    props.onClick?.();
  };
  return <Component {...props} onClick={onClick} />;
};

const TrackedButton = withLogOnClick(Button);
```

---

## 7. Memoization Rules (`useMemo` / `useCallback` / `React.memo`)

React compares objects, arrays, and functions **by reference**. New reference every render → memo comparisons fail. Memoize a value **only** when it is:

- a **dependency** of another hook (`useEffect`, `useMemo`, `useCallback`), **or**
- a prop passed to a component wrapped in `React.memo`, **or**
- passed down to something that meets one of the above.

Otherwise memoizing is just noise that slows reading.

```tsx
// ❌ pointless — onClick goes to a plain DOM node
const c = useCallback(() => {}, []);
return <button onClick={c} />;

// ✅ needed — submit is an effect dependency
const submit = useCallback(() => {}, []);
useEffect(() => { submit(); }, [submit]);
```

**`React.memo` is all-or-nothing — one unstable prop breaks it:**

```tsx
const ChildMemo = React.memo(Child);

// ❌ inline object + non-memoized children defeat the memo
<ChildMemo data={{ id: 1 }}>
  <Icon />
</ChildMemo>;

// ✅ every prop stable, including children
const data = useMemo(() => ({ id: 1 }), []);
const icon = useMemo(() => <Icon />, []);
<ChildMemo data={data}>{icon}</ChildMemo>;
```

- `useCallback` memoizes the **function**; `useMemo` memoizes the **result** of calling it.
- Don't spread `{...props}` into a memoized child — you can't guarantee every incoming prop is stable.
- `useMemo` for "expensive calculations" rarely matters — render/re-render cost usually dominates. **Measure first.**

---

## 8. Reconciliation & `key`

React diffs elements by **type + position** in the returned array. Same type & position → re-render (reuse instance + state). Type changes → unmount old, mount new.

```tsx
// ✅ stable id keeps identity across reorder/insert/remove
items.map((item) => <Row key={item.id} />);

// ❌ index as key — state/DOM misaligns when the list changes
items.map((item, i) => <Row key={i} />);
```

**`key` as a deliberate tool, even outside arrays:**

```tsx
// State reset: changing the key force-remounts (fresh state)
<Form key={userId} />;

// Force REUSE of the same instance across branches with one key
{isCompany
  ? <Input id="company-id" key="tax" />
  : <Input id="person-id" key="tax" />}
```

**Never define a component inside another component** — it's a brand-new type every render, so React remounts it and wipes its state on each parent render.

```tsx
// ❌ Input remounts (and loses focus/state) on every Parent render
const Parent = () => {
  const Input = () => <input />;
  return <Input />;
};

// ✅ define it outside
const Input = () => <input />;
const Parent = () => <Input />;
```

---

## 9. Context & Performance

Context lets you skip prop-drilling _and_ skip re-rendering the components in between. The catch: **every consumer re-renders when the provider value changes, and `React.memo` can't stop it.**

```tsx
// ❌ new value object every render → all consumers re-render
<Ctx.Provider value={{ isOpen, toggle }}>{children}</Ctx.Provider>;

// ✅ memoize the value
const value = useMemo(() => ({ isOpen, toggle }), [isOpen, toggle]);
<Ctx.Provider value={value}>{children}</Ctx.Provider>;
```

**Split state from API** so components that only fire actions never re-render on state changes:

```tsx
const DataCtx = createContext(null); // changes often
const ApiCtx = createContext(null);  // stable forever

const Provider = ({ children }) => {
  const [isOpen, setIsOpen] = useState(false);
  const data = useMemo(() => ({ isOpen }), [isOpen]);
  const api = useMemo(
    () => ({ open: () => setIsOpen(true), close: () => setIsOpen(false) }),
    [],
  );
  return (
    <DataCtx.Provider value={data}>
      <ApiCtx.Provider value={api}>{children}</ApiCtx.Provider>
    </DataCtx.Provider>
  );
};
```

No built-in selectors, but you can fake them with **HOC + `React.memo`** so only the slice you read triggers a re-render. For large apps, prefer an external store with real selectors.

---

## 10. Refs

A ref is a **mutable object whose `.current` survives re-renders and whose mutation does _not_ trigger one** (and is synchronous, unlike state).

```tsx
// Attach to a DOM node
const ref = useRef<HTMLInputElement>(null);
<input ref={ref} />; // ref.current is the node after mount

// Pass a ref through a component boundary
const Input = forwardRef<HTMLInputElement>((props, ref) => <input ref={ref} {...props} />);
// (React 19: `ref` is a normal prop, no forwardRef needed)

// Expose a curated imperative API, hiding internals
const Input = forwardRef((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    shake: () => {/* ... */},
  }), []);
  return <input ref={inputRef} />;
});
```

Use a ref for values that persist but aren't rendered: timer ids, previous values, latest-callback snapshots. Don't use it for anything the UI displays.

---

## 11. Closures & the Ref Escape Hatch

Every function created inside a component closes over the render's values. Memoize with a stale/missing dependency and you **freeze** those values — the classic "stale closure".

```tsx
// ❌ logs '' forever — closure frozen at mount, empty deps
const onClick = useCallback(() => console.log(value), []);

// ✅ stable callback that always reads the latest value
const ref = useRef<() => void>();
useEffect(() => {
  ref.current = () => console.log(value); // refreshed every render
});
const onClick = useCallback(() => ref.current?.(), []); // never changes
```

The pattern: keep a stable outer function, store the "live" logic in `ref.current`, refresh it in an effect. This is the foundation for the next pattern.

---

## 12. Debounce / Throttle Done Right

The debounced function must be created **once** (its internal timer must persist). Creating it in render re-creates the timer every render, so it never fires. Memoize the instance, and use the ref trick to keep access to fresh state.

```tsx
const useDebounce = (callback: () => void) => {
  const ref = useRef(callback);
  useEffect(() => { ref.current = callback; }); // always latest

  return useMemo(
    () => debounce(() => ref.current(), 1000), // created once
    [],
  );
};

const Input = () => {
  const [value, setValue] = useState("");
  const send = useDebounce(() => console.log(value)); // sees latest value
  return <input onChange={(e) => { setValue(e.target.value); send(); }} />;
};
```

```tsx
// ❌ new debounce (new timer) on every render → never debounces
const send = debounce(() => console.log(value), 1000);
```

---

## 13. `useLayoutEffect` for Flicker-Free Measurement

When you measure the DOM and immediately adjust it, `useEffect` runs **after** paint → the user sees the "before" frame flash. `useLayoutEffect` runs **synchronously before paint**, so only the final state is painted.

```tsx
const Menu = ({ items }) => {
  const ref = useRef<HTMLDivElement>(null);
  const [lastVisible, setLastVisible] = useState(-1);

  useLayoutEffect(() => {
    setLastVisible(getLastVisibleItem(ref.current)); // measure, then hide overflow
  }, [items]);

  return <div ref={ref}>{/* render based on lastVisible */}</div>;
};
```

**SSR caveat:** `useLayoutEffect` doesn't run on the server (React warns), so the first paint shows the unmeasured state. Gate the measured render behind a mounted flag or opt the feature out of SSR.

---

## 14. Portals

`createPortal` renders into a different DOM node while keeping the element in the React tree. Use it to free modals/tooltips/menus from `overflow: hidden` clipping and `z-index` stacking-context traps.

```tsx
import { createPortal } from "react-dom";

{isOpen && createPortal(<ModalDialog />, document.getElementById("root")!)}
```

Rule of thumb: **what happens in React stays in React** (context, re-renders, event bubbling follow the React tree), **what happens in the DOM follows the DOM** (CSS, native events, form boundaries follow the portal target). `position: fixed` escapes `overflow`, but **nothing escapes a stacking context** — portal out instead.

---

## 15. Data Fetching: Kill the Waterfalls

A waterfall is fetches that run in sequence because each waits on a parent to render/resolve. Fetch in parallel instead.

```tsx
// ❌ waterfall: Comments doesn't even mount until issue resolves
const Issue = () => {
  const { data: issue } = useData("/issue");
  if (!issue) return "loading";
  return <Comments />; // its fetch starts now, not earlier
};

// ✅ fire everything at once
const [sidebar, issue, comments] = await Promise.all([
  fetch("/sidebar").then((r) => r.json()),
  fetch("/issue").then((r) => r.json()),
  fetch("/comments").then((r) => r.json()),
]);
```

- Use `Promise.all` when you want them to land together; use independent `.then` chains to render each piece as it arrives.
- Watch the browser's ~6 parallel-request-per-host limit.
- Critical resources can be pre-fetched before React even boots.

---

## 16. Race Conditions in Fetching

When `id` changes mid-flight, a slow earlier response can overwrite a fast later one. Guard against it.

```tsx
// ✅ cleanest: cancel the stale request
useEffect(() => {
  const controller = new AbortController();
  fetch(`/page/${id}`, { signal: controller.signal })
    .then((r) => r.json())
    .then(setData)
    .catch((e) => { if (e.name !== "AbortError") throw e; });
  return () => controller.abort();
}, [id]);

// ✅ alternative: ignore the result of superseded effects
useEffect(() => {
  let active = true;
  fetch(`/page/${id}`)
    .then((r) => r.json())
    .then((r) => { if (active) setData(r); });
  return () => { active = false; };
}, [id]);
```

---

## 17. Universal Error Handling

Uncaught render errors after React 16 **unmount the whole app** — a few `ErrorBoundary`s in strategic places are non-negotiable.

```tsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error, info) { log(error, info); }
  render() {
    return this.state.hasError ? <ErrorScreen /> : this.props.children;
  }
}
```

**The split:** `ErrorBoundary` catches errors in the render tree below it but **misses async/callbacks**. `try/catch` catches async/callbacks but **misses nested-component render errors**. Bridge them by re-throwing async errors into render:

```tsx
const useThrowAsyncError = () => {
  const [, setState] = useState();
  return useCallback((error) => setState(() => { throw error; }), []);
};

const Component = () => {
  const throwAsyncError = useThrowAsyncError();
  useEffect(() => {
    fetch("/data").catch(throwAsyncError); // now the ErrorBoundary catches it
  }, [throwAsyncError]);
};
```

Or just use [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary), which packages this (`useErrorBoundary`) plus reset/fallback ergonomics.
