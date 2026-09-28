# React Anti-Pattern Catalog (v2)

The full reference of anti-patterns to flag, with default severities and fixes. Every finding
must cite one of these IDs. Read the entry before flagging so the justification is accurate.

Sources: *You Might Not Need an Effect* (react.dev/learn/you-might-not-need-an-effect),
*How to useMemo and useCallback* (developerway.com/posts/how-to-use-memo-use-callback),
*Separating Events from Effects* (react.dev/learn/separating-events-from-effects), and
*Updating Objects/Arrays in State* (react.dev/learn/updating-objects-in-state).

## Contents

- How to use this catalog
- Severity index (the single source of truth for default severities)
- Part A — State & Effects (A1–A12)
- Completing a data-fetching or mutation fix (applies to A4 and A10)
- Part B — Memoization (B1–B4)
- Part C — Component structure (C1)
- Part D — References, hook rules & state shape (D1–D9)
- Part E — Rendering, lists & context (E1–E3)
- Legitimate uses — do NOT flag these
- Judging severity

## How to use this catalog

Each entry gives: **Signal** (how to spot it) · **Default severity** · **Why it's a problem** ·
**How to fix**. Severity is a *default* — adjust it up or down for the concrete case and say why
when you do (see "Judging severity" at the bottom).

IDs are stable. New entries get new numbers; existing IDs are never renumbered, so older reviews
keep pointing at the right entry.

## Severity index (default)

This index is the only place where default severities are listed; SKILL.md and the report format
refer here rather than repeating the mapping.

**High — real bugs, crashes, or wrong behavior in production:**
- C1 — Component defined inside another component (remount wipes state/focus every render)
- D4 — Hook called conditionally / after an early return (breaks Rules of Hooks; React throws)
- D7 — Stale closure in a timer, async callback, or long-lived handler (reads an old value)
- D8 — Direct state mutation (React doesn't see the change; UI silently doesn't update)
- A10 — Data fetch in an Effect with no cleanup (race condition; stale results overwrite fresh)
- A11 — Effect that starts a timer/listener/subscription with no cleanup (leaks, duplicates)
- D3 — Missing hook dependencies (stale closures; effect silently skips re-runs)
- A4 — Event logic in an Effect (side effect like POST/notification fires on mount/reload, not on the action)
- A6 — App-init logic in an Effect with no guard (double-fires in Strict Mode; can invalidate tokens)
- A5 — Chains of Effects that trigger each other (cascade of renders; fragile, breaks on new requirements)
- E1 — `key={index}` or unstable keys on a dynamic list (state/input attaches to the wrong row) — *High when rows reorder/insert/remove and hold state; Low otherwise*

**Medium — extra renders, desync risk, fragile structure; correct today but a bug magnet:**
- A1 — Redundant state for a derived value, set from an Effect
- A2 — Derived data stored in state (filtered/transformed lists; prop-driven resets)
- A3 — Resetting all state on a prop change via an Effect (use `key` instead)
- A7 — Notifying the parent from an Effect (lift state up)
- A8 — Passing data up to the parent from an Effect (data should flow down)
- A9 — Subscribing to an external store via Effect + state (use `useSyncExternalStore`)
- A12 — Prop copied into state (`useState(props.x)`), so later prop updates are ignored
- D1 — New object/array/function literal used as a prop/dep that feeds a memo or dependency array
- D5 — Several coupled `useState`s whose updates entangle (consider `useReducer`)
- D6 — `useState` for a value never rendered (should be `useRef`)
- D9 — Several booleans modelling one mutually exclusive status (impossible states)
- E3 — Mega-context, or a provider `value` that is a fresh object every render

**Low — wasted work or cosmetic; no crash, no data loss:**
- B1 — Memoizing a prop passed to a non-`React.memo` component
- B2 — Memoizing a value passed to a DOM element
- B3 — Partial memoization of a `React.memo` component's props (one inline prop breaks it)
- B4 — Memoizing a cheap native JS operation (sort/filter/map/object creation)
- D2 — CSS-in-JS style object / class recomputed in the render body every render
- E2 — `{n && <X/>}` with a numeric left side renders a stray `0` — *High in React Native, where it crashes*

**Hard to spot.** C1, D7, D8, and A2-style desync look like working code and are the ones reviewers
miss most. Look for them actively rather than waiting for them to stand out.

Legitimate uses that must NOT be flagged are listed near the end. Check them before flagging
memoization, a `key` remount, `key={index}`, a prop-initialized state, or anything that looks
like mutation.

---

# Part A — State & Effects

Core rule: if a value can be computed from existing props or state, it does **not** belong in
state — compute it during render. State holds only *inputs* that can't be derived. For any
`useEffect`, ask: (1) Is there an external system? No external system → probably no Effect.
(2) Why does this run? If it runs because the user did something → **event handler**. If it runs
because the component was shown → an Effect is fine. (3) Does it start something? Then it must
stop it in a cleanup.

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

**Signal:** a `useState` holding something *computed from other state/props* — a filtered/transformed list, a full object copied out of a list, or a value you "reset"/"adjust" when a prop changes. Tell-tales: `setVisibleX`/`setFilteredX` keyed on source data, a `filtered`/`visible` state set from an `onChange`, `setSelectedUser(user)` storing a whole object from a list, or `setSelection(null)` in an Effect keyed on a prop.

```jsx
// 🔴 redundant state + Effect
const [filtered, setFiltered] = useState(items);
useEffect(() => { setFiltered(items.filter(i => i.name.includes(query))); }, [items, query]);

// ✅ derive during render — NOT state
const visible = items.filter(i => i.name.toLowerCase().includes(query.toLowerCase()));
```

For a selected item, or a value that must "reset" when a prop changes:

```jsx
// 🔴 full-object copy goes stale when `users` is refetched or edited
const [selectedUser, setSelectedUser] = useState(null);

// 🔴 Effect resets one slice of state on a prop change
useEffect(() => { setSelection(null); }, [items]);

// ✅ best: store the id, derive the object
const [selectedId, setSelectedId] = useState(null);
const selection = items.find(item => item.id === selectedId) ?? null;

// 🟡 fallback (only if it truly can't be derived): adjust during render, guarded
const [prevItems, setPrevItems] = useState(items);
if (items !== prevItems) { setPrevItems(items); setSelection(null); }
```

**Why:** the Effect version renders once with stale data, then re-renders after the Effect fires — two passes — and the two states can desync if `items` changes from outside (a refetch) before the Effect runs. A copied object keeps showing old field values after the source list updates. Deriving recomputes from current inputs every render, so it can never disagree.
**Fix:** keep only the input (often just an id) in state; derive the list/value during render. If it genuinely can't be reduced to an id/key, the render-time `setState` fallback needs the `items !== prevItems` guard (else infinite loop) and only state updates during render — no DOM writes or timers.

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
**Fix:** move the side effect into the event handler that causes it; delete any "trigger" state. **When the side effect is a network write (a POST/PUT/DELETE), moving it into the handler is only half the fix** — the handler still needs error handling and a loading/disabled state, and should guard against double-submits (disable the button or ignore while a request is in flight). Apply the checklist in "Completing a data-fetching or mutation fix" (the section right after A12, below).

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
**Fix:** use `useSyncExternalStore` (with a server-snapshot argument for SSR). If the Effect version also has **no cleanup**, that's A11 as well — report both.

### A10. Data fetching in an Effect *without cleanup* — *High*

**Signal:** a fetch Effect keyed on `query`/`page` that calls `setState` with no `AbortController` or `ignore` flag.

```jsx
// 🔴 race condition: responses can arrive out of order
useEffect(() => { fetchResults(query, page).then(json => setResults(json)); }, [query, page]);

// ✅ abort the stale request in cleanup
useEffect(() => {
  const controller = new AbortController();
  fetchResults(query, page, { signal: controller.signal })
    .then(json => setResults(json))
    .catch(e => { if (e.name !== 'AbortError') setError(e); });
  return () => controller.abort();
}, [query, page]);
```

**Why:** fetching *is* a valid Effect use — don't move it to a handler — but without cleanup a slow earlier response can land after a newer one and overwrite it with stale data.
**Fix:** the missing cleanup is the *flagged* defect, but cleanup alone is the floor, not a complete fix. A proper remediation covers four things — see "Completing a data-fetching or mutation fix" below. In short: use an `AbortController` (which actually cancels the request) rather than an `ignore` flag (which only discards the result), add error and loading state, debounce a high-frequency trigger, and reach for a reusable hook or a data library rather than hand-rolling it every time.

### A11. Effect that starts something and never stops it — *High*

**Signal:** a `useEffect` that calls `setInterval`, `setTimeout` (that sets state or has a visible effect), `addEventListener`, `.subscribe(...)`, opens a `WebSocket`/`EventSource`, creates a `ResizeObserver`/`IntersectionObserver`/`MutationObserver`, or initializes a third-party widget — and has **no `return` cleanup**, or a cleanup that doesn't undo what was started (e.g. `removeEventListener` with a different function reference).

```jsx
// 🔴 interval keeps running after unmount; Strict Mode starts two
useEffect(() => { setInterval(() => setCount(c => c + 1), 1000); }, []);

// 🔴 inline arrow — removeEventListener gets a different function, so nothing is removed
useEffect(() => {
  window.addEventListener('resize', () => setWidth(window.innerWidth));
  return () => window.removeEventListener('resize', () => setWidth(window.innerWidth));
}, []);

// ✅ keep a handle and undo exactly what you did
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);

useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize);
}, []);
```

**Why:** without cleanup the timer/listener outlives the component: memory leaks, state updates on unmounted components, and **duplicates** — every re-run of the Effect (dep change, Strict Mode's dev double-mount, remount via `key`) stacks another interval or handler, so things fire twice, three times, and so on.
**Fix:** return a cleanup that stops exactly what the Effect started (`clearInterval`, `clearTimeout`, `removeEventListener` with the *same* function, `unsubscribe()`, `observer.disconnect()`, `socket.close()`, `widget.destroy()`). For browser/3rd-party *values* mirrored into state, prefer `useSyncExternalStore` (A9). Lower to Medium for a one-shot `setTimeout` whose only effect is harmless after unmount.

### A12. Prop copied into state — *Medium*

**Signal:** `const [x, setX] = useState(props.x)` (or `useState(x)` where `x` is a prop) where the prop is **not** named or documented as an initial value, and the parent can change it later. Often paired with a "sync" Effect `useEffect(() => setX(props.x), [props.x])` — that pair is this entry plus A2/A3.

```jsx
// 🔴 title ignores every later change to props.title
function EditableTitle({ title }) {
  const [draft, setDraft] = useState(title);
  return <input value={draft} onChange={e => setDraft(e.target.value)} />;
}

// ✅ fully controlled: the parent owns the value
function EditableTitle({ title, onTitleChange }) {
  return <input value={title} onChange={e => onTitleChange(e.target.value)} />;
}

// ✅ or explicitly uncontrolled: name it as a seed and reset with a key
function EditableTitle({ initialTitle }) {
  const [draft, setDraft] = useState(initialTitle);
  /* ... */
}
<EditableTitle key={docId} initialTitle={doc.title} />
```

**Why:** `useState(initial)` reads its argument only on the first render. When the parent later passes a new value, the component keeps showing the old one — a silent desync that looks like "the prop doesn't work." Adding a sync Effect to patch it creates the double-render and drift problems of A2/A3.
**Fix:** pick one owner. Either make the component controlled (use the prop directly, report changes up), or make the "initial" nature explicit (`initialX`/`defaultX`) and reset with `key` when the identity changes. If the value is purely derivable from the prop, compute it during render (A1/A2).

---

### Completing a data-fetching or mutation fix — apply this to A10 and A4 (fetches/POSTs)

When a finding's fix involves fetching data or performing a mutation (a POST/PUT/DELETE), don't stop
the recommendation at "add a cleanup" or "move the call into the handler." That removes the
flagged anti-pattern but leaves the code half-built. A complete fix — and therefore a complete
recommendation in the review — addresses:

1. **Cancellation.** Use an `AbortController` rather than a bare `ignore` flag. The flag only
   *ignores* a stale response; `AbortController` actually *aborts* the request, freeing the
   connection and stopping wasted work. Abort in the Effect cleanup (or on a new submit for a mutation).
2. **Error handling.** A `.catch`/`try-catch` that sets an `error` state, so a failed or rejected
   request surfaces to the user instead of becoming an unhandled rejection or a silent no-op.
   Ignore `AbortError` — it's the expected result of cancellation, not a failure.
3. **Loading, error and empty UI.** A `loading`/`isPending` flag so the UI can show a spinner or
   skeleton, a rendered error branch, and an empty branch — not just the happy path
   (`data.map(...)` on `undefined` crashes). For mutations, disable the submit control while the
   request is in flight (which also prevents double-submits). Model these as one status, not
   several booleans (D9). Judge this by what the JSX renders, not by which variables exist: an
   `isError` that is never set, or a loading flag while the render still dereferences `data`
   unguarded, means the piece is missing.
4. **Debounce/throttle a high-frequency trigger.** When the fetch is keyed on something that changes
   rapidly (a search box firing on every keystroke), debounce so you send one request per pause, not
   one per character. This also shrinks the race window.

**Don't hand-roll all four inline on every call site.** The right move is usually to extract a
reusable hook (`useData(url)` / `useFetch`) or adopt a data-fetching library — **TanStack Query
(React Query)**, **SWR**, or **RTK Query** — which give caching, cancellation, and error/loading
state out of the box. `axios` covers cancellation and error ergonomics if a full library is overkill.
Present the library as an option that fits a project with many fetches, not as the only right answer.

Illustrative reusable hook (cancellation + loading + error in one place):

```jsx
function useFetch(url) {
  const [state, setState] = useState({ status: 'loading', data: null, error: null });
  useEffect(() => {
    const controller = new AbortController();
    setState({ status: 'loading', data: null, error: null });
    fetch(url, { signal: controller.signal })
      .then((r) => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then((data) => setState({ status: 'success', data, error: null }))
      .catch((error) => {
        if (error.name !== 'AbortError') setState({ status: 'error', data: null, error });
      });
    return () => controller.abort();
  }, [url]);
  return state;
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

**React Compiler check first.** If the project uses React Compiler (see "Legitimate uses"), the
compiler memoizes automatically. Do not report B1–B4 in that case; at most add one line noting that
manual memoization is redundant there.

### B1. Memoizing props passed to a NON-memoized component — *Low*

**Signal:** a `useCallback`/`useMemo` value passed to a child that is **not** wrapped in `React.memo`, and not used in any dependency array downstream.

```jsx
// 🔴 useless: Item re-renders with the parent regardless of props
const onClick = useCallback(() => { /* ... */ }, []);   // remove this
return data.map((d) => <Item item={d} onClick={onClick} />);
```

**Why:** a component re-renders when its parent does, regardless of whether props changed. Memoizing a prop does nothing unless the receiver is memoized or the value feeds a dependency array.
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

**Why:** `Input` is a brand-new function on every re-render, and functions are never referentially equal. React compares the old `type` to the new `type`, sees two different functions, and **unmounts the old tree and mounts a new one** instead of re-rendering in place. That remount wipes **local state and focus on every keystroke/re-render**, tears down and rebuilds DOM (visible flicker), and re-runs Effects. One of the biggest performance/correctness killers in React, and hard to trace because the code *looks* fine. React Compiler does not fix this.
**Fix:** declare the component at module scope (or its own file) and pass render-time values via props. (Calling a plain render helper as a function — `{renderRow(item)}` — is not this anti-pattern; only rendering it as `<X />` is.)

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

**Why:** object/array/function literals are never referentially equal between renders. If the receiver is memoized or the value feeds a dependency array, the fresh reference defeats it every time. This is the flip side of Part B — B removes memo that does nothing; D1 is the case where a stable reference actually matters. (A context provider's `value` is the most expensive version of this — see E3.)
**Fix:** lift truly-constant values to module scope; use `useState`/`useMemo` (per Part B's rules) only when the value must live across renders but can't be a module constant. If nothing downstream compares the reference, severity is Low. Not a finding under React Compiler.

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

// ✅ list every referenced value; make the request cancelable
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/items/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setData)
    .catch(e => { if (e.name !== 'AbortError') setError(e); });
  return () => controller.abort();
}, [id]);
```

**Why:** an incomplete dependency array captures stale values and silently skips re-runs.
**Fix:** list every referenced value; fix the *cause* of the lint warning rather than silencing `exhaustive-deps`. Options, in order: add the dep; move a function the Effect uses *inside* the Effect; stabilize the value (module scope, D1). If the Effect reads a value it must see fresh but should **not** re-run for (e.g. logging the current `theme` when a connection opens), extract that logic into a `useEffectEvent` (React 19.2+) and leave it out of the deps. On older React, use a latest-value ref (see D7). Pairs with A10 (the cleanup half of doing Effect fetches correctly) — apply the full checklist when the Effect fetches.

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
**Fix:** call all hooks at the top level unconditionally, then branch. (`use(promise)`/`use(Context)` in React 19 *may* be called conditionally — it is the one exception.)

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
**Fix:** consolidate the coupled fields into a `useReducer`. Reach for it when the *transitions* are coupled, not merely because there are several `useState`s — many independent form fields are fine as separate state or one object. When the coupled fields are booleans describing a single status, that's D9.

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

### D7. Stale closure in a timer, async callback, or long-lived handler — *High*

**Signal:** a callback created in one render and run later — inside `setTimeout`/`setInterval`, after an `await`/`.then`, in a subscription/listener registered once, or in a handler stored somewhere long-lived — that reads state or props **as they were when it was created**. Typical shapes:
- `setInterval(() => setCount(count + 1), 1000)` in an Effect with `[]` deps (count stays 0 → 1 forever).
- `setTimeout(() => alert(count), 3000)` in a click handler when the intent is the *current* count at fire time.
- `await save(); setItems([...items, saved])` — `items` may have changed during the await.

```jsx
// 🔴 next state computed from a captured value
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []);

// ✅ 1. computing next state → functional updater, no captured value needed
const id = setInterval(() => setCount(c => c + 1), 1000);

// 🔴 reads count at click time, not at fire time
const handleDelayedAlert = () => { setTimeout(() => alert(`Count: ${count}`), 3000); };

// ✅ 2. reading the latest value later → latest-value ref (update it in an Effect, not during render)
const countRef = useRef(count);
useEffect(() => { countRef.current = count; });
const handleDelayedAlert = () => { setTimeout(() => alert(`Count: ${countRef.current}`), 3000); };

// ✅ 3. the stale read is inside an Effect → useEffectEvent (React 19.2+)
const onTick = useEffectEvent(() => { log(count); });
useEffect(() => { const id = setInterval(onTick, 1000); return () => clearInterval(id); }, []);
```

**Why:** each render has its own props and state; a function created during a render keeps seeing that render's values forever. When it runs later, it acts on old data — lost increments, wrong alerts, list updates that overwrite concurrent changes. It's hard to spot because the code reads correctly.
**Fix:** pick by situation — (1) computing the next state from the previous one: functional updater (`setX(prev => …)`); (2) reading the latest value inside an async callback created by an event handler: latest-value ref; (3) inside an Effect, where the value shouldn't cause a re-run: `useEffectEvent` on React 19.2+, otherwise the ref pattern. If the value *should* restart the Effect, it's D3 instead — add it to the deps. Check the React version before recommending `useEffectEvent`.

**Deciding intent (event handlers).** In an Effect or a long-lived subscription, a stale read is
almost always a bug. In an event handler, reading the value *as of the click* is often exactly what
the user expects, so the closure is only a bug when the code shows it wants the value at fire time:
a name like `latest`/`current`, a comment, a value it then writes back to state, or a read after an
`await` that it merges with state. If the intent is ambiguous (e.g. `setTimeout(() => alert(count),
3000)` in a click handler), don't report a High finding. Mention it as "worth a look, not certain"
with a one-line explanation of both readings, and don't give it a plan step. A wrong High here sends
the user off to "fix" code that may be correct.

### D8. Direct state mutation — *High*

**Signal:** modifying an object/array that is (or is inside) state or props, then setting it back or relying on a re-render. Look for: property assignment on a state object (`user.name = …`, `todo.done = !todo.done` on an item found with `.find`), in-place array methods on state or props (`push`, `pop`, `splice`, `shift`, `unshift`, `sort`, `reverse`, `fill`, `copyWithin`), `delete obj.key`, `Object.assign(stateObj, …)`, nested spread that stops one level too early, and `setX(x)` right after changing `x`. **`props.items.sort()` in render mutates the parent's data** — treat that as this entry too.

```jsx
// 🔴 same reference → React bails out, no re-render; also corrupts any memo/history
const todo = todos.find(t => t.id === id);
todo.done = !todo.done;
setTodos(todos);

// 🔴 sort() is in place — mutates state (or the parent's prop)
const sorted = items.sort(byName);

// 🔴 shallow copy, nested object still shared
setUser({ ...user, address: user.address }); user.address.city = 'Varna';

// ✅ new references along the changed path
setTodos(prev => prev.map(t => t.id === id ? { ...t, done: !t.done } : t));
const sorted = items.toSorted(byName);            // or [...items].sort(byName)
setUser(prev => ({ ...prev, address: { ...prev.address, city: 'Varna' } }));
```

**Why:** React decides whether to re-render by comparing references (`Object.is`). Mutating in place keeps the reference the same, so the UI may not update, or updates later at a random re-render. It also breaks `React.memo`, dependency arrays, undo/history, and React Compiler's assumptions, and mutating props changes data the parent still owns.
**Fix:** produce a new object/array for every level you change — `map`/`filter`/spread, or the non-mutating ES2023 methods (`toSorted`, `toReversed`, `toSpliced`, `with`). For deep updates, use Immer (`produce`/`useImmer`) or flatten the state shape.

### D9. Several booleans modelling one status — *Medium*

**Signal:** two or more `useState<boolean>` (or boolean fields) that describe *mutually exclusive* phases of the same thing — `isLoading` / `isError` / `isSuccess` / `isRetrying`, `isOpen` / `isClosing` — and code that has to set several of them at once to keep them consistent.

```jsx
// 🔴 16 combinations, most impossible (loading AND error AND success?)
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);

// ✅ one status; data/error only exist in the states that have them
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: Error };
const [state, setState] = useState<State>({ status: 'idle' });
```

**Why:** N booleans allow 2^N combinations, but the UI only has a few valid states. Forgetting one setter leaves the component "loading and errored" at once; render code grows chains of `&&` checks that disagree with each other.
**Fix:** replace them with a single status value (a string union, or a discriminated union that carries `data`/`error`), or a reducer when transitions carry rules (D5). Independent booleans (`isDarkMode` and `isSidebarOpen`) are not this anti-pattern.

---

# Part E — Rendering, lists & context

### E1. `key={index}` or unstable keys on a dynamic list — *High when rows reorder/insert/remove and hold state; Low otherwise*

**Signal:** `items.map((item, i) => <Row key={i} … />)` where the list can be reordered, filtered, sorted, or have items inserted/removed anywhere but the end — **especially** when rows contain inputs, local state, animations, or uncontrolled components. Also: keys generated during render (`key={Math.random()}`, `key={crypto.randomUUID()}`, `key={Date.now()}`), and keys that aren't unique among siblings.

```jsx
// 🔴 delete row 0 → row 1's typed text now shows in what was row 0
{todos.map((todo, index) => <TodoItem key={index} todo={todo} />)}

// 🔴 new key every render → every row remounts every render (state and focus lost)
{todos.map(todo => <TodoItem key={Math.random()} todo={todo} />)}

// ✅ stable, unique id from the data
{todos.map(todo => <TodoItem key={todo.id} todo={todo} />)}
```

**Why:** React matches list children between renders by key. With index keys, removing or reordering items shifts which data each key points at, so component state, input values, and focus attach to the wrong row. Random keys make every row a new component on every render — full remount, the same damage as C1.
**Fix:** use a stable id from the data. If the data has none, assign one when the item is *created* (not during render). Random keys generated in render are always High. Index keys on a static, never-reordered list are fine (see "Legitimate uses").

### E2. `&&` with a numeric left side — *Low (High in React Native)*

**Signal:** `{count && <X/>}`, `{items.length && <List/>}`, `{user.unread && …}` where the left side is a number (or could be `NaN`).

```jsx
// 🔴 renders "0" when count is 0
{count && <CartBadge count={count} />}
{items.length && <List items={items} />}

// ✅ make the condition a real boolean
{count > 0 && <CartBadge count={count} />}
{items.length > 0 ? <List items={items} /> : null}
```

**Why:** `&&` returns the left operand when it's falsy. `false`, `null`, and `undefined` render nothing, but `0` and `NaN` render as text. On the web that's a stray "0" in the UI. In React Native, a bare number outside `<Text>` throws ("Text strings must be rendered within a `<Text>` component"), so raise to High there.
**Fix:** compare explicitly (`> 0`), coerce (`!!count` / `Boolean(count)`), or use a ternary with `null`.

### E3. Mega-context, or an unstable provider `value` — *Medium*

**Signal:** either (a) one context carrying several unrelated, independently changing values (`{ user, cart, theme, notifications }`), or (b) a provider whose `value` is a fresh object/array/function created during render — `value={{ user, setUser }}` — in a component that re-renders for other reasons.

```jsx
// 🔴 (a) any change to cart re-renders every theme consumer, and vice versa
<AppContext.Provider value={{ user, cart, theme, notifications }}>

// 🔴 (b) new object every render → every consumer re-renders every time App does
function App() {
  const [user, setUser] = useState(null);
  return <UserContext.Provider value={{ user, setUser }}>{children}</UserContext.Provider>;
}

// ✅ split by concern, and stabilize each value
const value = useMemo(() => ({ user, setUser }), [user]);
<UserContext.Provider value={value}>…</UserContext.Provider>

// ✅ for frequently read state + rarely changing actions, split them
<CartStateContext value={cart}><CartDispatchContext value={dispatch}>…
```

**Why:** every component that calls `useContext(Ctx)` re-renders whenever the provider's `value` changes by reference. A catch-all context turns every small update into an app-wide re-render; an unmemoized value does the same even when nothing inside it changed.
**Fix:** split contexts by concern (and state vs. dispatch); memoize the `value` (unnecessary under React Compiler). For large, frequently updated shared state, a store with selectors (`useSyncExternalStore`, Zustand, Redux) lets consumers subscribe to only the slice they read. Lower to Low when the value changes rarely and the consumer tree is small.

---

# Legitimate uses — do NOT flag these

Check these before flagging, to avoid false positives.

- **Anything memo-related when React Compiler is on.** If `babel-plugin-react-compiler` (or the
  framework's switch, e.g. `reactCompiler: true` in `next.config`) is configured, manual
  `useMemo`/`useCallback`/`React.memo` is redundant but harmless. Don't report B1–B4, D1, or E3(b).
  The compiler does **not** fix C1, D3, D4, D7, D8, E1, or A-series logic — keep flagging those.
  A component opted out with `"use no memo"` is reviewed as if there were no compiler.
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
  subscribing to a genuinely external system *with cleanup*) is a correct Effect. Only event-caused
  logic (A4) belongs in handlers.
- **`key={index}` on a static list** that is never reordered, filtered, or edited, and whose rows
  hold no state (a fixed nav menu, rendered markdown paragraphs). Not a finding.
- **Prop-initialized state that is explicitly a seed** — the prop is named `initialX`/`defaultX`,
  or documented as the starting value — especially when the parent resets it with `key`. That's the
  intended uncontrolled pattern, not A12.
- **"Mutation" that isn't touching state:** writes to an Immer draft inside `produce`/`useImmer`,
  mutating a freshly created local copy before setting it (`const next = [...items]; next.sort();
  setItems(next)`), mutating `ref.current`, or building a new object in a local variable during
  render. Not D8.
- **The latest-value ref pattern** (`ref.current = value` in an Effect, read in an async callback)
  is the D7 fix. Don't flag that ref as D6.
- **Capturing a value on purpose.** A timeout that should use the value *at the moment of the
  click* (e.g. "send to the recipient selected when you pressed Send") is correct closure
  behavior, not D7. Flag only when the code clearly wants the latest value; see "Deciding intent"
  under D7.
- **`&&` with a boolean left side** (`isOpen &&`, `items.length > 0 &&`, `!!count &&`) is fine.
  Only numeric or possibly-`NaN` left sides are E2.

# Judging severity

Start from the default severity in the severity index, then adjust for the concrete case and state
your reasoning in the finding:

- Raise severity when the anti-pattern is on a hot path, handles money/auth, fires network writes,
  runs in React Native (E2), or the desync/stale window is user-visible.
- Lower it when the code is trivial, the list is tiny or static, the value is primitive, or nothing
  downstream consumes the unstable reference.
- If something matches a signal but falls under "Legitimate uses," it is **not a finding** — don't
  report it (or, if it's a close call, note briefly why you decided it's fine).