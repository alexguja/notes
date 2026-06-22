# React Patterns Cheatsheet

Distilled from Nadia Makarevich's [_Advanced React_](https://www.advanced-react.com). Patterns use ✅ for recommended and ❌ for anti-patterns. The throughline is **performance via composition**, not memoization — most sections walk a naive version → why it breaks → the fix.

---

## 1. The Re-render Mental Model

1. **State update is the only source of re-renders.** Props changing does nothing on its own.
2. **Re-renders cascade _down_, never up or sideways.** When a component re-renders, React re-renders every component it renders inside, regardless of props.
3. **A re-render is just React calling your function again.** Cheap by default; the cost is the depth of the subtree and what each function does.
4. **A component re-renders when its element object changes** (`Object.is` on the element before vs after). Same reference → React bails out of that subtree.
5. **Reach for composition before memoization.** `React.memo` is the last resort, not the first.

```tsx
// A state update in App re-renders App AND everything below it,
// even components with no props at all.
const App = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsOpen(true)} />
      <VerySlowComponent /> {/* re-renders on every toggle, for no reason */}
    </>
  );
};
```

**The big myth: "a component re-renders when its props change."** False without `React.memo`. A component re-renders because its parent did. And only state can drive it — mutating a plain variable is invisible to React:

```tsx
// ❌ never updates the UI — a plain variable doesn't trigger a re-render
let isOpen = false;
<Button onClick={() => (isOpen = true)} />;

// ✅ state update schedules a re-render
const [isOpen, setIsOpen] = useState(false);
<Button onClick={() => setIsOpen(true)} />;
```

---

## 2. State Lives Where It's Used (Move State Down)

If only a small part of the tree uses a piece of state, push that state — and the components that read it — into the smallest possible child, so expensive siblings sit outside the re-render zone.

```tsx
// ❌ isOpen lives in App, so the whole app re-renders on every toggle
const App = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <div>
      <Button onClick={() => setIsOpen(true)}>Open</Button>
      {isOpen && <ModalDialog onClose={() => setIsOpen(false)} />}
      <VerySlowComponent />
      <BunchOfStuff />
    </div>
  );
};

// ✅ state isolated; the slow siblings never re-render on toggle
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
    <BunchOfStuff />
  </div>
);
```

### The custom-hooks trap

State hidden in a hook still belongs to the component that calls the hook. The hook can return nothing and it still re-renders its host on every state change — and that propagates through chains of hooks.

```tsx
// ❌ App re-renders on every window resize, even though the width is never used
const useResizeDetector = () => {
  const [width, setWidth] = useState(0);
  useEffect(() => {
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize);
  }, []);
  return null; // returns nothing — but the state is still here
};

const useModal = () => {
  const [isOpen, setIsOpen] = useState(false);
  useResizeDetector(); // hidden state from a nested hook leaks in
  return { isOpen, open: () => setIsOpen(true), close: () => setIsOpen(false) };
};

const App = () => {
  const { isOpen, open, close } = useModal(); // App now re-renders on resize
  return (
    <div>
      <Button onClick={open} />
      {isOpen && <ModalDialog onClose={close} />}
      <VerySlowComponent />
    </div>
  );
};
```

**Fix:** extract the hook into the small component that owns it (same "move state down" idea), so `VerySlowComponent` is untouched.

---

## 3. Components/Children as Props

When state _must_ live in a parent (scroll position, mouse coords), pass the slow components _in_ as `children` or an element prop. Elements passed as props are created by the **caller**, so the parent re-rendering doesn't change their reference → React skips them.

```tsx
// ❌ MovingBlock needs `position` state in the parent, so everything
//    inside re-renders 60×/sec while scrolling — janky
const MainScrollableArea = () => {
  const [position, setPosition] = useState(300);
  return (
    <div onScroll={(e) => setPosition(getPosition(e.currentTarget.scrollTop))}>
      <MovingBlock position={position} />
      <VerySlowComponent />
      <BunchOfStuff />
    </div>
  );
};

// ✅ the slow tree is created in App and passed through as `children`;
//    its reference is stable, so scrolling doesn't re-render it
const ScrollableArea = ({ children }) => {
  const [position, setPosition] = useState(300);
  return (
    <div onScroll={(e) => setPosition(getPosition(e.currentTarget.scrollTop))}>
      <MovingBlock position={position} />
      {children}
    </div>
  );
};

const App = () => (
  <ScrollableArea>
    <VerySlowComponent />
    <BunchOfStuff />
  </ScrollableArea>
);
```

`children` is just a prop — `<Parent><Child/></Parent>` is identical to `<Parent children={<Child/>} />`. Why it works: `ScrollableArea` re-renders on scroll, compares the `children` prop before/after, finds the same object reference (it was created in `App`, which didn't re-render), and skips that subtree.

---

## 4. Elements as Props (Flexible Configuration)

When a component sprouts endless config props (`iconColor`, `iconSize`, `iconName`, `isIconAvatar`...), pass the element instead and hand control to the consumer.

```tsx
// ❌ props explosion — every icon variation needs new props
const Button = ({ iconName, iconColor, iconSize, isIconAvatar /* ...20 more */ }) => { /* ... */ };

// ✅ consumer owns the icon entirely
const Button = ({ icon }: { icon: ReactElement }) => <button>Submit {icon}</button>;

<Button icon={<Loading />} />;
<Button icon={<Error color="red" size="large" />} />;
<Button icon={<Avatar />} />;
```

Same idea scales to whole layouts (modals, lists) via multiple element props:

```tsx
const Modal = ({ children, footer }) => (
  <div className="modal">
    <div className="content">{children}</div>
    <div className="footer">{footer}</div>
  </div>
);

<Modal footer={<SubmitButton />}>
  <SomeForm />
</Modal>;
```

**Conditional rendering of an element prop is free.** Creating an element is just making an object — it doesn't render until the component that holds it actually renders:

```tsx
const App = () => {
  const footer = <Footer />; // cheap object; Footer does NOT render here
  return isDialogOpen ? <ModalDialog footer={footer} /> : null; // renders only when open
};
```

**Defaults via `cloneElement`** (fragile — keep to trivial cases). Always merge so consumer props win:

```tsx
// ✅ defaults, but the consumer can still override
const Button = ({ appearance, size, icon }) => {
  const defaults = { size, color: appearance === "primary" ? "white" : "black" };
  return <button>Submit {React.cloneElement(icon, { ...defaults, ...icon.props })}</button>;
};

// ❌ defaults only — silently throws away <Loading color="red" />
React.cloneElement(icon, defaults);
```

---

## 5. Render Props

Use when the parent needs to pass its **own state/props into** the element it receives — something plain elements-as-props can't do.

```tsx
// ❌ elements as props: parent can't feed state to the icon
const Button = ({ icon }) => <button>Submit {icon}</button>;

// ✅ render prop: parent passes computed props + state into the function
const Button = ({ appearance, size, renderIcon }) => {
  const [isHovered, setIsHovered] = useState(false);
  const iconProps = { size, color: appearance === "primary" ? "white" : "black", isHovered };
  return (
    <button onMouseEnter={() => setIsHovered(true)} onMouseLeave={() => setIsHovered(false)}>
      Submit {renderIcon(iconProps)}
    </button>
  );
};

<Button renderIcon={(props) => <HomeIcon {...props} highlighted={props.isHovered} />} />;
```

**Children as a render prop** shares stateful logic without lifting it up:

```tsx
const ScrollDetector = ({ children }: { children: (n: number) => ReactElement }) => {
  const [scroll, setScroll] = useState(0);
  return <div onScroll={(e) => setScroll(e.currentTarget.scrollTop)}>{children(scroll)}</div>;
};

<ScrollDetector>{(scroll) => (scroll > 30 ? <StickyNav /> : null)}</ScrollDetector>;
```

> Hooks replaced render props for ~99% of stateful-logic sharing (`useResizeDetector()` reads cleaner than a wrapper). Render props still earn their keep for **configuration flexibility** (the icon case) and when the logic is **tied to a specific DOM element** (scroll, intersection observer).

---

## 6. Higher-Order Components

A function that takes a component and returns an enhanced one. Hooks cover most cases now, but HOCs still shine for cross-cutting wrappers: injecting props, intercepting events, lifecycle hooks.

```tsx
// canonical shape
const withSomething = (Component) => (props) => {
  // inject logic/props here
  return <Component {...props} extra="data" />;
};

// inject + intercept a callback
const withLogOnClick = (Component) => (props) => {
  const onClick = () => {
    console.log("clicked");
    props.onClick?.();
  };
  return <Component {...props} onClick={onClick} />;
};

// add lifecycle behaviour
const withMountLog = (Component) => (props) => {
  useEffect(() => {
    console.log("mounted");
  }, []);
  return <Component {...props} />;
};

const TrackedButton = withLogOnClick(Button);
```

Pass extra config via the factory argument or an extra prop. Downside: "wrapper hell" in DevTools — prefer hooks unless you specifically need to wrap a component without touching it.

---

## 7. Memoization Rules (`useMemo` / `useCallback` / `React.memo`)

React compares objects, arrays, and functions **by reference**. A new reference every render → memo comparisons fail.

```tsx
const useCallbackish = (fn, deps) => (depsEqual(deps) ? cached : (cached = fn)); // memoizes the fn
const useMemoish = (fn, deps) => (depsEqual(deps) ? cached : (cached = fn())); // memoizes the result
```

Memoize a value **only** when it is: a **dependency** of another hook; **or** a prop passed to a `React.memo` component; **or** passed down to something that meets one of those. Otherwise it's noise that costs memory and reading effort.

```tsx
// ❌ pointless — onClick goes to a plain DOM node, and the component
//    re-renders anyway when its parent does
const onClick = useCallback(() => {}, []);
return <button onClick={onClick} />;

// ✅ needed — submit is an effect dependency
const submit = useCallback(() => {}, []);
useEffect(() => { submit(); }, [submit]);
```

### `React.memo` is all-or-nothing — one unstable prop breaks it

`React.memo` skips a re-render only if **every** prop is referentially equal. A single inline object, function, spread, or `children` element defeats it.

```tsx
const ChildMemo = React.memo(Child);

// ❌ new object + new children element every render → memo never holds
<ChildMemo data={{ id: 1 }}>
  <Icon />
</ChildMemo>;

// ✅ every prop stable, including children
const data = useMemo(() => ({ id: 1 }), []);
const icon = useMemo(() => <Icon />, []);
<ChildMemo data={data}>{icon}</ChildMemo>;
```

Two especially sneaky breakers:

```tsx
// ❌ spread — you can't guarantee every incoming prop is stable
const Wrapper = (props) => <ChildMemo {...props} />;

// ❌ a non-memoized function from a custom hook
const useForm = () => ({ submit: () => {} }); // new submit every render
const { submit } = useForm();
<ChildMemo onChange={submit} />; // memo always re-renders
// ✅ memoize inside the hook: const submit = useCallback(() => {}, [])
```

- `useCallback` memoizes the **function**; `useMemo` memoizes the **result** of calling it.
- `useMemo` for "expensive calculations" rarely matters — sorting 300 items might be ~2ms while re-rendering the list is ~20ms. **Measure first**, then fix re-renders (move state down / memo the list items), not the maths.

---

## 8. Reconciliation & `key`

React diffs elements by **type + position** in the returned array. Same type & position → it **reuses the instance and its state**. Type changes → unmount old, mount new.

### The "mysterious bug": state leaks across a conditional

```tsx
// ❌ typing in one input, then toggling, keeps the text — both branches are
//    an <Input/> at the same position, so React reuses the same instance
const Form = () => {
  const [isCompany, setIsCompany] = useState(false);
  return (
    <>
      <Checkbox onChange={() => setIsCompany((v) => !v)} />
      {isCompany ? <Input id="company-tax-id" /> : <Input id="person-tax-id" />}
    </>
  );
};

// ✅ distinct keys tell React these are different elements → remount, fresh state
{isCompany ? (
  <Input id="company-tax-id" key="company" />
) : (
  <Input id="person-tax-id" key="person" />
)}
```

### Dynamic arrays need stable keys

```tsx
// ❌ index as key — when the list reorders, state/DOM stays at the old position
items.map((item, i) => <Row key={i} defaultValue={item.label} />);

// ✅ stable id — identity (and any internal state) travels with the row
items.map((item) => <Row key={item.id} defaultValue={item.label} />);
```

Index keys are only safe for **static** lists that never reorder, change length, or get filtered. With `React.memo`'d rows, a wrong key can also force needless re-renders when items shift.

### `key` as a deliberate tool

```tsx
// State reset: changing the key force-remounts with fresh state
<Form key={userId} />;       // new user → blank form
<Article key={articleId} />; // route change → reset scroll/draft

// Force REUSE of one instance across branches by giving both the same key
{isCompany ? <Input id="company" key="tax" /> : <Input id="person" key="tax" />}
```

Remounting is more expensive than re-rendering — use key-based reset deliberately, not in hot loops.

### Never define a component inside another component

A nested definition is a brand-new function (new `type`) every render, so React unmounts and remounts it — wiping state, losing focus, flashing the UI.

```tsx
// ❌ Input remounts on every Parent render; its state + focus are destroyed
const Parent = () => {
  const Input = () => <input />;
  return <Input />;
};

// ✅ define it at module scope and pass data via props
const Input = () => <input />;
const Parent = () => <Input />;
```

---

## 9. Context & Performance

Context skips prop-drilling _and_ skips re-rendering the components in between. The catch: **every consumer re-renders when the provider value changes, and `React.memo` can't stop it.**

```tsx
// ❌ new value object every render → all consumers re-render
<Ctx.Provider value={{ isOpen, toggle }}>{children}</Ctx.Provider>;

// ✅ memoize the value
const value = useMemo(() => ({ isOpen, toggle }), [isOpen, toggle]);
<Ctx.Provider value={value}>{children}</Ctx.Provider>;
```

**Split state from API** so components that only fire actions never re-render when the state changes. `useReducer` makes the API genuinely dependency-free:

```tsx
const DataCtx = createContext(null); // changes often
const ApiCtx = createContext(null);  // stable forever

const Provider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, { isOpen: false });

  const data = useMemo(() => state, [state]);
  const api = useMemo(
    () => ({
      open: () => dispatch({ type: "open" }),
      close: () => dispatch({ type: "close" }),
    }),
    [], // dispatch is stable, so the API never changes
  );

  return (
    <DataCtx.Provider value={data}>
      <ApiCtx.Provider value={api}>{children}</ApiCtx.Provider>
    </DataCtx.Provider>
  );
};

// a component that only opens the nav never re-renders when isOpen flips
const OpenButton = () => {
  const { open } = useContext(ApiCtx);
  return <button onClick={open}>Open</button>;
};
```

No built-in selectors, but you can fake one with **HOC + `React.memo`**: the wrapper reads context and re-renders, but the memoized inner component only re-renders if the slice it actually receives changed.

```tsx
const withOpen = (Component) => {
  const Memo = React.memo(Component);
  return (props) => {
    const { open } = useContext(ApiCtx); // stable
    return <Memo {...props} open={open} />;
  };
};
```

For large apps, prefer an external store (Redux/Zustand) with real selectors.

---

## 10. Refs

A ref is a **mutable object whose `.current` survives re-renders and whose mutation does _not_ trigger one** (and is synchronous, unlike state).

```tsx
// ❌ ref for rendered data — the UI never updates (no re-render)
const ref = useRef(0);
return <p>{ref.current}</p>; // always 0

// ✅ ref for things the UI doesn't display: timer ids, previous values,
//    latest-callback snapshots, and DOM nodes
const inputRef = useRef<HTMLInputElement>(null);
<input ref={inputRef} />; // inputRef.current is the node after mount
```

Pass a ref to a child's DOM node, then expose a curated imperative API instead of leaking internals:

```tsx
const Input = forwardRef<HTMLInputElement>((props, ref) => <input ref={ref} {...props} />);
// (React 19: `ref` is a normal prop — forwardRef no longer required)

const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    shake: () => {/* ... */},
  }), []);
  return <input ref={inputRef} />;
});

// parent: inputRef.current.focus() / .shake() — implementation stays hidden
```

Read/write refs in effects and event handlers, not during render.

---

## 11. Closures & the Ref Escape Hatch

Every function created inside a component closes over that render's values. Memoize it with a stale/missing dependency and you **freeze** those values — the classic "stale closure".

```tsx
// ❌ logs '' forever — the closure was frozen at mount with empty deps
const [value, setValue] = useState("");
const onClick = useCallback(() => console.log(value), []);
```

The fix is either a correct dependency array (`[value]`) — or, when you need a **stable** callback (e.g. a prop to a `React.memo` child) that still reads fresh values, the **ref escape hatch**: keep the live logic in `ref.current`, refresh it in an effect, call it from a stable wrapper.

```tsx
const ref = useRef<() => void>();
useEffect(() => {
  ref.current = () => console.log(value); // refreshed after every render
});
const onClick = useCallback(() => ref.current?.(), []); // stable, never changes
```

This keeps `onClick` referentially stable (so memoized children don't re-render) while `ref.current` always points at the latest closure. It's the foundation of the next pattern.

---

## 12. Debounce / Throttle Done Right

Debounce waits until calls stop; throttle fires at most once per interval. Both rely on a **persistent timer**, so the debounced function must be created **once** — and it must still see the latest state.

```tsx
// ❌ new debounce (new timer) every render → never actually debounces
const send = debounce(() => console.log(value), 1000);

// ❌ memoized, but the callback is a stale closure; adding `value` to its deps
//    re-creates the debounce on every keystroke, defeating the point
const send = useMemo(() => debounce(() => console.log(value), 1000), []);
```

✅ Combine the ref escape hatch with a once-created debounce:

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
  const send = useDebounce(() => console.log(value)); // sees the latest value
  return <input onChange={(e) => { setValue(e.target.value); send(); }} />;
};
```

---

## 13. `useLayoutEffect` for Flicker-Free Measurement

When you measure the DOM and immediately adjust it, `useEffect` runs **after** paint, so the user sees the "before" frame flash. `useLayoutEffect` runs **synchronously before paint** — the browser treats render + measure + re-render as one un-painted task, so only the final state shows.

```tsx
// Responsive menu: render all items, measure, then collapse overflow into "More"
const Menu = ({ items }) => {
  const ref = useRef<HTMLDivElement>(null);
  const [lastVisible, setLastVisible] = useState(-1);

  useLayoutEffect(() => {
    setLastVisible(getLastVisibleIndex(ref.current)); // measure + hide before paint
  }, [items]);

  return (
    <div ref={ref}>
      {items.filter((_, i) => lastVisible === -1 || i <= lastVisible).map((item) => (
        <a key={item.id} href={item.href}>{item.name}</a>
      ))}
      {lastVisible < items.length - 1 && <button>More</button>}
    </div>
  );
};
```

With `useEffect` you'd get: paint all items → effect → re-paint collapsed = visible flash.

**SSR caveat:** `useLayoutEffect` doesn't run on the server (React warns), so the first paint shows the unmeasured state. Gate the measured render behind a client-mounted flag in the parent so the hook order stays stable:

```tsx
const Nav = (props) => {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? <Menu {...props} /> : <MenuPlaceholder />;
};
```

---

## 14. Portals

`createPortal` renders into a different DOM node while keeping the element in the React tree. Use it to free modals/tooltips/menus from `overflow: hidden` clipping and `z-index` **stacking-context** traps.

```tsx
import { createPortal } from "react-dom";

// ❌ even z-index: 9999 can't lift the modal above a sticky header:
//    the header creates a stacking context that traps everything in <main>
<header className="sticky" />
<main>
  {isOpen && <Modal style={{ zIndex: 9999 }} />}
</main>;

// ✅ portal the modal to the root — now it's a sibling of the header, untrapped
<main>
  {isOpen && createPortal(<Modal />, document.getElementById("root")!)}
</main>;
```

Rule of thumb: **what happens in React stays in React** — context, re-renders, and event bubbling follow the React tree (a click inside a portalled modal still bubbles to its React parent). **What happens in the DOM follows the DOM** — CSS inheritance, native `addEventListener`, and `<form>` submit boundaries follow the portal target, not the React parent. `position: fixed` escapes `overflow`, but **nothing escapes a stacking context** — portal out instead.

---

## 15. Data Fetching: Kill the Waterfalls

A waterfall is fetches that run in sequence because each waits on a parent to render/resolve. With three 1s/2s/3s requests, a waterfall takes 6s; in parallel it takes 3s.

```tsx
// ❌ waterfall: Comments doesn't even mount until issue resolves,
//    and issue doesn't fetch until sidebar renders
const Issue = () => {
  const { data: issue } = useData("/issue");
  if (!issue) return "loading";
  return <Comments />; // its fetch starts only now
};

// ✅ fire everything at once, render when all land
const useAllData = () => {
  const [state, setState] = useState({ sidebar: null, issue: null, comments: null });
  useEffect(() => {
    Promise.all([fetch("/sidebar"), fetch("/issue"), fetch("/comments")])
      .then((rs) => Promise.all(rs.map((r) => r.json())))
      .then(([sidebar, issue, comments]) => setState({ sidebar, issue, comments }));
  }, []);
  return state;
};
```

Variations:
- **Independent `.then` chains** (no `Promise.all`) render each piece the moment it arrives — progressive UI, at the cost of multiple re-renders.
- **Data-provider Contexts** wrap the app so every request fires in parallel before any leaf mounts, while leaves stay simple (`useContext`).
- Watch the browser's **~6 parallel-requests-per-host** limit; critical resources can even be pre-fetched before React boots.

---

## 16. Race Conditions in Fetching

When the dependency (`id`) changes mid-flight, a slow earlier response can overwrite a fast later one. Four fixes, roughly in order of preference:

```tsx
// ✅ 1. AbortController — actually cancel the stale request
useEffect(() => {
  const controller = new AbortController();
  fetch(`/page/${id}`, { signal: controller.signal })
    .then((r) => r.json())
    .then(setData)
    .catch((e) => { if (e.name !== "AbortError") throw e; });
  return () => controller.abort();
}, [id]);

// ✅ 2. Ignore superseded results via the effect cleanup closure
useEffect(() => {
  let active = true;
  fetch(`/page/${id}`).then((r) => r.json()).then((r) => { if (active) setData(r); });
  return () => { active = false; };
}, [id]);

// ✅ 3. Compare the result against the latest requested id (needs an id in the response)
const idRef = useRef(id);
useEffect(() => {
  idRef.current = id;
  fetch(`/page/${id}`).then((r) => r.json()).then((r) => { if (r.id === idRef.current) setData(r); });
}, [id]);

// ✅ 4. Force a remount with key so the stale component (and its setState) is gone
<Page id={id} key={id} />;
```

`key` is the bluntest — it throws away all component state (focus, refs), so reserve it for when a full reset is what you actually want.

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

**The split, and why neither alone is enough:**

- `try/catch` catches errors in callbacks and awaited promises, but **not** errors thrown by nested components during render (`<Child/>` is just an object, not a call), and you can't wrap `useEffect` or the component's return in it.
- `ErrorBoundary` catches anything thrown in the render tree below it, but **misses** async errors and event-handler errors.

Bridge them by catching async errors and **re-throwing them into render** via a state updater, so the boundary can see them:

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

Or use [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary), which packages this (`useErrorBoundary`) plus reset/fallback ergonomics.
