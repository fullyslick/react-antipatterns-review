# React Anti-Pattern Catalog

The full reference of anti-patterns to flag, with default severities and fixes. Every finding
must cite one of these IDs. Read the entry before flagging so the justification is accurate.

Sources: *You Might Not Need an Effect* (react.dev/learn/you-might-not-need-an-effect) and
*How to useMemo and useCallback* (developerway.com/posts/how-to-use-memo-use-callback).

## How to use this catalog

Each entry gives: **Signal** (how to spot it) · **Default severity** · **Why it's a problem** ·
**How to fix**. Severity is a *default* — adjust it up or down for the concrete case and say why
when you do (see "Judging severity" at the bottom).

## Severity index (default)

**High — real bugs, crashes, or wrong behavior in production:**
- C1 — Component defined inside another component (remount wipes state/focus every render)
- D4 — Hook called conditionally / after an early return (breaks Rules of Hooks; React throws)
- A10 — Data fetch in an Effect with no cleanup (race condition; stale results overwrite fresh)
- D3 — Missing hook dependencies (stale closures; effect silently skips re-runs)
- A4 — Event logic in an Effect (side effect like POST/notification fires on mount/reload, not on the action)
- A6 — App-init logic in an Effect with no guard (double-fires in Strict Mode; can invalidate tokens)
- A5 — Chains of Effects that trigger each other (cascade of renders; fragile, breaks on new requirements)
  **Medium — extra renders, desync risk, fragile structure; correct today but a bug magnet:**
- A1 — Redundant state for a derived value, set from an Effect
- A2 — Derived data stored in state (filtered/transformed lists; prop-driven resets)
- A3 — Resetting all state on a prop change via an Effect (use `key` instead)
- A7 — Notifying the parent from an Effect (lift state up)
- A8 — Passing data up to the parent from an Effect (data should flow down)
- A9 — Subscribing to an external store via Effect + state (use `useSyncExternalStore`)
- D1 — New object/array/function literal used as a prop/dep that feeds a memo or dependency array
- D5 — Several coupled `useState`s whose updates entangle (consider `useReducer`)
- D6 — `useState` for a value never rendered (should be `useRef`)
  **Low — wasted work, no user-visible bug; clean-up, not correctness:**
- B1 — Memoizing a prop passed to a non-`React.memo` component
- B2 — Memoizing a value passed to a DOM element
- B3 — Partial memoization of a `React.memo` component's props (one inline prop breaks it)
- B4 — Memoizing a cheap native JS operation (sort/filter/map/object creation)
- D2 — CSS-in-JS style object / class recomputed in the render body every render
  Legitimate uses that must NOT be flagged are listed at the very end. Check them before flagging
  memoization or a `key` remount.

---

# Part A — State & Effects

Core rule: if a value can be computed from existing props or state, it does **not** belong in
state — compute it during render. State holds only *inputs* that can't be derived. For any
`useEffect`, ask: (1) Is there an external system? No external system → probably no Effect.
(2) Why does this run? If it runs because the user did something → **event handler**. If it runs
because the component was shown → an Effect is fine.

### A1. Redundant state for derived values — *Medium*

**Signal:** a `useState` whose setter appears **only** inside a `useEffect` that reads other state/props.

```jsx
// 🔴 redundant state + Effect
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(firstName + ' ' + lastName); }, [firstName, lastName]);
 
// ✅ calculate during render
const fullName = firstName + ' ' + lastName;
```

**Why:** an extra render pass with a stale value, then an immediate re-render — plus a chance for the two states to drift out of sync.
**Fix:** delete the state and the Effect; compute the value as a plain `const` during render.

### A2. Derived data doesn't belong in state — *Medium*

**Signal:** a `useState` holding something *computed from other state/props* — a filtered/transformed list, or a value you "reset"/"adjust" when a prop changes. Tell-tales: `setVisibleX`/`setFilteredX` keyed on source data, a `filtered`/`visible` state set from an `onChange`, or `setSelection(null)` in an Effect keyed on a prop.

```jsx
// 🔴 redundant state + Effect
const [filtered, setFiltered] = useState(items);
useEffect(() => { setFiltered(items.filter(i => i.name.includes(query))); }, [items, query]);
 
// ✅ derive during render — NOT state
const visible = items.filter(i => i.name.toLowerCase().includes(query.toLowerCase()));
```

For a value that must "reset" when a prop changes (e.g. clearing a selection):

```jsx
// 🔴 Effect resets one slice of state on a prop change
useEffect(() => { setSelection(null); }, [items]);
 
// ✅ best: store the id, derive the object
const [selectedId, setSelectedId] = useState(null);
const selection = items.find(item => item.id === selectedId) ?? null;
 
// 🟡 fallback (only if it truly can't be derived): adjust during render, guarded
const [prevItems, setPrevItems] = useState(items);
if (items !== prevItems) { setPrevItems(items); setSelection(null); }
```

**Why:** the Effect version renders once with stale data, then re-renders after the Effect fires — two passes — and the two states can desync if `items` changes from outside (a refetch) before the Effect runs. Deriving recomputes from current inputs every render, so it can never disagree.
**Fix:** keep only the input in state; derive the list/value during render. If it genuinely can't be reduced to an id/key, the render-time `setState` fallback needs the `items !== prevItems` guard (else infinite loop) and only state updates during render — no DOM writes or timers.

### A3. Resetting all state on a prop change via Effect — *Medium*

**Signal:** `useEffect(() => { setX(''); }, [someId])` — clearing state when an id/prop flips.

```jsx
// 🔴
useEffect(() => { setComment(''); }, [userId]);
 
// ✅ give the subtree a key instead
<Profile userId={userId} key={userId} />
```

**Why:** renders once with stale state, then again — and you'd have to repeat it for every nested piece of state. A `key` change tells React to recreate the component and reset *all* its (and its children's) state automatically.
**Fix:** pass `key={id}` to the subtree you want to reset instead of clearing fields in an Effect. (This is the *intended reset* case; see the note under "Legitimate uses" about when a `key` remount is itself the smell.)

### A4. Event logic living in an Effect — *High*

**Signal:** an Effect that fires a notification, POST, navigation, or analytics that is conceptually a *reaction to a user action*, keyed on a flag like `isInCart` or a `jsonToSubmit` holder.

```jsx
// 🔴 fires again on every reload where isInCart is already true
useEffect(() => { if (product.isInCart) showNotification(`Added ${product.name}!`); }, [product]);
 
// ✅ share logic via a function called from the handlers
function buyProduct() { addToCart(product); showNotification(`Added ${product.name}!`); }
 
// 🔴 "trigger" state: setJsonToSubmit just to make an Effect fire the POST
const [jsonToSubmit, setJsonToSubmit] = useState(null);
useEffect(() => { if (jsonToSubmit) post('/api/register', jsonToSubmit); }, [jsonToSubmit]);
 
// ✅ POST directly in the handler
function handleSubmit(e) { e.preventDefault(); post('/api/register', { firstName, lastName }); }
```

**Why:** logic caused by a specific interaction belongs in the handler. In an Effect it re-fires whenever the dependency happens to be set — on mount, on reload, on unrelated re-renders — causing duplicate POSTs, double notifications, etc. Keep in an Effect only what runs *because the component was displayed* (e.g. a `visit_form` analytics event on mount).
**Fix:** move the side effect into the event handler that causes it; delete any "trigger" state. **When the side effect is a network write (a POST/PUT/DELETE), moving it into the handler is only half the fix** — the handler still needs error handling and a loading/disabled state, and should guard against double-submits (disable the button or ignore while a request is in flight). Apply the four-point checklist in "Completing a data-fetching or mutation fix" (above, under A10): cancellation, error state, loading state, and — for a real form — prefer a reusable submit hook or a data/mutation library (TanStack Query's `useMutation`, etc.) over a hand-rolled fire-and-forget call.

### A5. Chains of Effects that trigger each other — *High*

**Signal:** multiple Effects where each one's dependency is a state var that a *previous* Effect sets — a cascade `setA` → Effect → `setB` → Effect → `setC`.

```jsx
// 🔴 each set triggers the next Effect; N re-renders, fragile
useEffect(() => { if (card?.gold) setGoldCardCount(c => c + 1); }, [card]);
useEffect(() => { if (goldCardCount > 3) { setRound(r => r+1); setGoldCardCount(0); } }, [goldCardCount]);
 
// ✅ derive what you can; compute the next state in the one handler that caused it
const isGameOver = round > 5;
function handlePlaceCard(nextCard) { /* set card, count, round together here */ }
```

**Why:** a re-render between every link, and the chain breaks the moment a new requirement (e.g. "step through history") sets state to a past value. **Exception:** legitimate when each step *synchronizes with the network* (e.g. dependent dropdowns fetching options).
**Fix:** derive what you can; compute the next state explicitly inside the single handler that caused the change (`const nextRound = round + 1;`).

### A6. App-init logic in an Effect — *High*

**Signal:** `useEffect(() => { checkAuthToken(); loadFromLocalStorage(); }, [])` in a top-level component.

```jsx
// ✅ guard so it survives dev double-mount
let didInit = false;
useEffect(() => { if (!didInit) { didInit = true; /* one-time init */ } }, []);
 
// ✅ or run at module scope, before render
if (typeof window !== 'undefined') { checkAuthToken(); loadDataFromLocalStorage(); }
```

**Why:** Effects run twice in dev (Strict Mode) — un-guarded one-time init can double-fire and, e.g., invalidate an auth token.
**Fix:** guard with a module-scope `didInit` flag, or run the init at module scope (root modules only).

### A7. Notifying the parent from an Effect — *Medium*

**Signal:** `useEffect(() => { onChange(isOn); }, [isOn])`.

```jsx
// 🔴 onChange runs a render too late; two render passes
useEffect(() => { onChange(isOn); }, [isOn, onChange]);
 
// ✅ update both in the same event
function updateToggle(next) { setIsOn(next); onChange(next); }
 
// ✅✅ or remove the state entirely — lift it up, fully controlled
function Toggle({ isOn, onChange }) { /* no local state */ }
```

**Why:** the parent learns a render too late, and you get two passes. Whenever two components keep state in sync, **lift state up**.
**Fix:** call `onChange` in the same handler that sets local state, or lift the state to the parent and make the component controlled.

### A8. Passing data *up* to the parent in an Effect — *Medium*

**Signal:** a child fetches/derives data and pushes it to the parent via `useEffect(() => onFetched(data), [data])`.

```jsx
// 🔴 child → parent data flow is hard to trace
function Child({ onFetched }) { const data = useSomeAPI(); useEffect(() => { if (data) onFetched(data); }, [data]); }
 
// ✅ fetch in the parent, pass down
function Parent() { const data = useSomeAPI(); return <Child data={data} />; }
```

**Why:** data should flow **down**. Child-updates-parent-in-Effect makes bugs untraceable.
**Fix:** lift the fetch/derivation to the parent and pass the result down as props.

### A9. Subscribing to an external store with an Effect — *Medium*

**Signal:** an Effect that `addEventListener`s a browser/3rd-party source and mirrors it into `useState`.

```jsx
// 🟡 works, but error-prone
useEffect(() => { /* add/remove online/offline listeners, setIsOnline */ }, []);
 
// ✅ purpose-built hook
useSyncExternalStore(subscribe, () => navigator.onLine, () => true);
```

**Why:** manually syncing a mutable external source into state is error-prone (missed unsubscribes, SSR mismatches).
**Fix:** use `useSyncExternalStore` (with a server-snapshot argument for SSR).

### A10. Data fetching in an Effect *without cleanup* — *High*

**Signal:** a fetch Effect keyed on `query`/`page` that calls `setState` with no `ignore` flag or `AbortController`.

```jsx
// 🔴 race condition: responses can arrive out of order
useEffect(() => { fetchResults(query, page).then(json => setResults(json)); }, [query, page]);
 
// ✅ ignore stale responses in cleanup
useEffect(() => {
  let ignore = false;
  fetchResults(query, page).then(json => { if (!ignore) setResults(json); });
  return () => { ignore = true; };
}, [query, page]);
```

**Why:** fetching *is* a valid Effect use — don't move it to a handler — but without cleanup a slow earlier response can land after a newer one and overwrite it with stale data.
**Fix:** the missing cleanup is the *flagged* defect, but a bare `ignore` flag is the floor, not a complete fix. A proper remediation of a fetch Effect covers four things — see "Completing a data-fetching or mutation fix" directly below. In short: prefer an `AbortController` (which actually cancels the in-flight request) over an `ignore` flag (which only discards its result), add error and loading state, debounce a high-frequency trigger, and reach for a reusable hook or a data library rather than hand-rolling it every time.
 
---

### Completing a data-fetching or mutation fix — apply this to A10 and A4 (fetches/POSTs)

When a finding's fix involves fetching data or performing a mutation (a POST/PUT/DELETE), don't stop
the recommendation at "add a cleanup flag" or "move the call into the handler." That removes the
flagged anti-pattern but leaves the code half-built. A complete fix — and therefore a complete
recommendation in the review — addresses:

1. **Cancellation.** Prefer an `AbortController` over a bare `ignore` flag. The flag only *ignores*
   a stale response; `AbortController` actually *aborts* the request, freeing the connection and
   stopping wasted work. Abort in the Effect cleanup (or on a new submit for a mutation).
2. **Error handling.** A `.catch`/`try-catch` that sets an `error` state, so a failed or rejected
   request surfaces to the user instead of becoming an unhandled rejection or a silent no-op.
3. **Loading state.** A `loading`/`isPending` flag so the UI can show a spinner or skeleton and, for
   mutations, disable the submit control while the request is in flight (which also prevents
   double-submits).
4. **Debounce/throttle a high-frequency trigger.** When the fetch is keyed on something that changes
   rapidly (a search box firing on every keystroke), debounce so you send one request per pause, not
   one per character. This also shrinks the race window.
   **Don't hand-roll all four inline on every call site.** The right move is usually to extract a
   reusable hook (`useData(url)` / `useFetch`) or adopt a data-fetching library — **TanStack Query
   (React Query)**, **SWR**, or **RTK Query** — which give caching, cancellation, and error/loading
   state out of the box. `axios` covers cancellation and error ergonomics if a full library is overkill.
   Reserve the minimal `ignore`-flag version for the simplest one-off fetch.

Illustrative reusable hook (cancellation + loading + error in one place):

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    fetch(url, { signal: controller.signal })
      .then((r) => { if (!r.ok) throw new Error('Network error'); return r.json(); })
      .then(setData)
      .catch((e) => { if (e.name !== 'AbortError') setError(e); })
      .finally(() => setLoading(false));
    return () => controller.abort();
  }, [url]);
  return { data, loading, error };
}
```

In the review, when you flag A10 or the A4 POST case, say the fix "isn't just X" and name which of
the four pieces are missing in *this* code (e.g. "no error/loading state, and the ignore flag should
be an AbortController"), then point at the hook/library approach. A one-line "add cleanup" is an
incomplete recommendation.
 
---

# Part B — Memoization (`useMemo` / `useCallback`)

Core rule: `useMemo`/`useCallback` do exactly one thing — preserve a reference **between
re-renders**. They help *only* when that reference is actually compared somewhere (a `React.memo`
boundary or a hook dependency array). If it isn't feeding a real reference comparison, remove it.
All four below are **Low** severity by default — they cost a little and do nothing, but they don't
break behavior.

### B1. Memoizing props passed to a NON-memoized component — *Low*

**Signal:** a `useCallback`/`useMemo` value passed to a child that is **not** wrapped in `React.memo`.

```jsx
// 🔴 useless: Item re-renders with the parent regardless of props
const onClick = useCallback(() => { /* ... */ }, []);   // remove this
return data.map((d) => <Item item={d} onClick={onClick} />);
```

**Why:** a component re-renders when its parent does, regardless of whether props changed. Memoizing a prop does nothing unless the receiver is memoized.
**Fix:** drop the `useCallback`/`useMemo`; use a plain function/value.

### B2. Memoizing props passed to a DOM element — *Low*

**Signal:** the memoized value is passed (directly or through a chain) as an attribute to a native DOM element (`<button>`, `<div>`, `<input>`, …).

```jsx
// 🔴 useless: onClick goes straight to a DOM node
const onClick = useCallback(() => { /* ... */ }, []);   // remove this
return <button onClick={onClick}>Click me</button>;
```

**Why:** DOM elements are never "memoized components," so handlers/values passed to them gain nothing.
**Fix:** drop the memoization.

### B3. Partial memoization of a `React.memo` component's props — *Low*

**Signal:** a `React.memo` component receives **at least one** non-memoized prop (an inline object/array/function literal, or a value not itself memoized).

```jsx
const PageMemoized = React.memo(Page);
const onClick = useCallback(() => { /* ... */ }, []);
// 🔴 value={[1,2,3]} is a fresh array each render → PageMemoized re-renders anyway → onClick's useCallback is wasted too
return <PageMemoized onClick={onClick} value={[1, 2, 3]} />;
```

**Why:** `React.memo` skips a re-render only if **every** prop compares equal. One non-memoized prop breaks the comparison and makes the *other* memoized props pointless.
**Fix:** memoize *all* props (including hoisting/`useMemo`-ing the inline object/array), or remove memoization from *all* of them.

### B4. Memoizing cheap "native" JavaScript operations — *Low*

**Signal:** `useMemo` wrapping a sort / filter / map / object or array creation / `new Date()`.

```jsx
// 🔴 useless: sorting is not the bottleneck; rendering is
const sortedCountries = useMemo(() => orderBy(countries, 'name', sort), [countries, sort]);
 
// ✅ if optimizing at all, memoize the expensive part: the render tree
const content = useMemo(() => orderBy(countries, 'name', sort).map((c) => <Item country={c} key={c.id} />), [countries, sort]);
```

**Why:** these ops are effectively free vs. rendering (sorting ~250 items <2ms; rendering it ~20ms). The bottleneck is child re-renders, not the array method.
**Fix:** remove the `useMemo`. Only if there's genuinely heavy CPU work, memoize the **rendered JSX subtree**, not the array op. Reach for it only after composition / `React.memo` placement / state colocation — and measure first.
 
---

# Part C — Component structure

### C1. Defining a component inside another component — *High*

**Signal:** a `const X = () => …` (or `function X()`) component declaration in the *body* of another component, then rendered as `<X />`.

```jsx
// 🔴 Input is redefined on every render of Component
const Component = () => {
  const Input = () => <input />;
  return <Input />;
};
 
// ✅ hoist it out to module scope (pass data via props)
const Input = () => <input />;
const Component = () => <Input />;
```

**Why:** `Input` is a brand-new function on every re-render, and functions are never referentially equal. React compares the old `type` to the new `type`, sees two different functions, and **unmounts the old tree and mounts a new one** instead of re-rendering in place. That remount wipes **local state and focus on every keystroke/re-render**, tears down and rebuilds DOM (visible flicker), and re-runs Effects. One of the biggest performance/correctness killers in React, and hard to trace because the code *looks* fine.
**Fix:** declare the component at module scope (or its own file) and pass render-time values via props.
 
---

# Part D — References, hook rules & state shape

### D1. New object/array/function literal used as a "value" every render — *Medium (Low if nothing consumes the reference)*

**Signal:** a plain `const x = { … }` / `[ … ]` / `() => …` in the component body, passed as a prop or used as a hook dependency — a fresh reference on every render.

```jsx
// 🔴 new reference every render → breaks memo/deps downstream
function Parent() { const value = { flag: true }; return <Child value={value} />; }
 
// ✅ module-scope constant if it never changes
const value = { flag: true };
function Parent() { return <Child value={value} />; }
 
// ✅ stable state ref if it can change over time
const [value] = useState({ flag: true });
```

**Why:** object/array/function literals are never referentially equal between renders. If the receiver is memoized or the value feeds a dependency array, the fresh reference defeats it every time. This is the flip side of Part B — B removes memo that does nothing; D1 is the case where a stable reference actually matters.
**Fix:** lift truly-constant values to module scope; use `useState`/`useMemo` (per Part B's rules) only when the value must live across renders but can't be a module constant. If nothing downstream compares the reference, severity is Low.

### D2. CSS-in-JS style object/class created inside the component — *Low*

**Signal:** an `emotion`/`styled` `css({...})` (or inline style object) computed in the render body on every render.

```jsx
// 🔴 style recomputed on every render
function Box() { return <div className={css({ background: 'tomato', width: '100%' })} />; }
 
// ✅ define once at module scope, reuse
const box = css({ background: 'tomato', width: '100%' });
function Box() { return <div className={box} />; }
```

**Why:** static styles don't depend on props/state, so recomputing them each render is wasted work (and, for some CSS-in-JS runtimes, repeated serialization/injection).
**Fix:** hoist anything static to module scope; keep only genuinely dynamic style bits inside render.

### D3. Missing hook dependencies — *High*

**Signal:** a value (prop, state, or function) read inside `useEffect`/`useMemo`/`useCallback` but absent from the dependency array — often an empty `[]` on an Effect that uses `id`, `query`, etc.

```jsx
// 🔴 uses `id` but deps are [] → stale fetch, never re-runs on id change
useEffect(() => { fetch(`/api/items/${id}`).then(r => r.json()).then(setData); }, []);
 
// ✅ list every referenced value; make the logic cancelable
useEffect(() => {
  let cancelled = false;
  fetch(`/api/items/${id}`).then(r => r.json()).then(d => { if (!cancelled) setData(d); });
  return () => { cancelled = true; };
}, [id]);
```

**Why:** an incomplete dependency array captures stale values and silently skips re-runs.
**Fix:** list every referenced value; fix the *cause* of the lint warning (stabilize the dep, move it, or restructure) rather than silencing `exhaustive-deps`. Pairs with A10 (the cleanup half of doing Effect fetches correctly).

### D4. Calling hooks conditionally or after an early return — *High*

**Signal:** a `useState`/`useEffect`/any hook inside an `if`, loop, `&&`, or below an early `return`.

```jsx
// 🔴 hook order changes when `ready` flips → breaks the Rules of Hooks
function Panel({ ready }) {
  if (!ready) return null;
  const [open, setOpen] = useState(false);
  return <button onClick={() => setOpen(o => !o)}>{String(open)}</button>;
}
 
// ✅ all hooks first, unconditionally; branch after
function Panel({ ready }) {
  const [open, setOpen] = useState(false);
  if (!ready) return null;
  return <button onClick={() => setOpen(o => !o)}>{String(open)}</button>;
}
```

**Why:** React identifies hooks by call order, which must be identical on every render. A conditional hook shifts that order, corrupts state associations, and React throws.
**Fix:** call all hooks at the top level unconditionally, then branch.

### D5. Exploding `useState` where `useReducer` fits — *Medium*

**Signal:** several related `useState`s whose updates are entangled (change one → must adjust the others), scattered across the component.

```jsx
// 🔴 tangled, easy to leave inconsistent
const [text, setText] = useState('');
const [error, setError] = useState('');
const [touched, setTouched] = useState(false);
 
// ✅ centralize the transitions in a reducer
const [state, dispatch] = useReducer(reducer, undefined, getInitial);
```

**Why:** when several state fields change together, independent setters invite inconsistent intermediate states and duplicated update logic. A reducer makes each transition a single, testable, atomic move.
**Fix:** consolidate the coupled fields into a `useReducer`. Reach for it when the *transitions* are coupled, not merely because there are several `useState`s.

### D6. `useState` for a value that isn't rendered (should be `useRef`) — *Medium*

**Signal:** a `useState` flag/counter/handle that never appears in JSX and exists only to gate logic (a "has run" flag, a timer id, a previous-value cache) — each update forces a needless re-render.

```jsx
// 🔴 re-renders just to flip a flag nothing renders
const [ran, setRan] = useState(false);
useEffect(() => { if (!ran) { doSomething(); setRan(true); } }, [ran]);
 
// ✅ a ref holds mutable non-UI data without re-rendering
const ranRef = useRef(false);
useEffect(() => { if (!ranRef.current) { ranRef.current = true; doSomething(); } }, []);
```

**Why:** state exists to drive the UI; changing it re-renders. If a value never influences what's painted, keep it in a `useRef`.
**Fix:** move the non-UI value to `useRef`. (Inverse mistake: using a ref for something the UI *does* depend on, which then won't update on screen — don't do that either.)
 
---

# Legitimate uses — do NOT flag these

Check these before flagging, to avoid false positives.

- **Memoization that feeds a hook dependency downstream.** If a non-primitive value ends up in a
  `useEffect`/`useMemo`/`useCallback` dependency array — even several props deep — it *needs* a
  stable reference, or the downstream hook fires every render. Keep it.
- **Memoization where the receiver is `React.memo` AND every prop is memoized.** Then the stable
  reference lets `React.memo` actually skip the re-render. Keep it.
- **Memoizing genuinely expensive CPU work**, or a heavy rendered JSX subtree. Keep it (but a
  comment noting the measured cost is a plus).
- **Fetching in an Effect *with* cleanup** (A10 done right) is correct — don't flag it as "logic in
  an Effect." Fetching syncs to current query/page; it belongs in an Effect.
- **`key`-based remount when the intent is to reset** all state for a new identity (new `userId`,
  a "start over" flow) is the *correct* fix (see A3), not an anti-pattern. It's only a smell when
  used to paper over a state-sync bug on a component you actually want to keep mounted — there,
  derive during render (A1/A2) instead of forcing a remount.
- **An Effect that runs because the component was shown** (mount analytics like `visit_form`,
  subscribing to a genuinely external system) is a correct Effect. Only event-caused logic (A4)
  belongs in handlers.
# Judging severity

Start from the default severity in each entry, then adjust for the concrete case and state your
reasoning in the finding:

- Raise severity when the anti-pattern is on a hot path, handles money/auth, fires network writes,
  or the desync/stale window is user-visible.
- Lower it when the code is trivial, the list is tiny, the value is primitive, or nothing downstream
  consumes the unstable reference.
- If something matches a signal but falls under "Legitimate uses," it is **not a finding** — don't
  report it (or, if it's a close call, note briefly why you decided it's fine).