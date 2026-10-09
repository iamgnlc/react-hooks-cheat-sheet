# React 19 Hooks Cheat Sheet

A quick reference to every built-in hook in React 19 (up to 19.2), with a short explanation and a TypeScript example for each.

> All hooks follow the **Rules of Hooks**: call them only at the top level of a component or custom hook, never inside loops, conditions or nested functions. The one exception is `use`, which *can* be called conditionally.

## Index

1. [State hooks](#state-hooks)
   - [`useState`](#usestate)
   - [`useReducer`](#usereducer)
2. [Context hook](#context-hook)
   - [`useContext`](#usecontext)
3. [Ref hooks](#ref-hooks)
   - [`useRef`](#useref)
   - [`useImperativeHandle`](#useimperativehandle)
4. [Effect hooks](#effect-hooks)
   - [`useEffect`](#useeffect)
   - [`useLayoutEffect`](#uselayouteffect)
   - [`useInsertionEffect`](#useinsertioneffect)
   - [`useEffectEvent`](#useeffectevent-react-192) *(19.2)*
5. [Performance hooks](#performance-hooks)
   - [`useMemo`](#usememo)
   - [`useCallback`](#usecallback)
   - [`useTransition`](#usetransition)
   - [`useDeferredValue`](#usedeferredvalue)
6. [Actions & forms hooks](#actions--forms-hooks-new-in-react-19) *(new in 19)*
   - [`useActionState`](#useactionstate)
   - [`useOptimistic`](#useoptimistic)
   - [`useFormStatus`](#useformstatus-from-react-dom)
7. [Resource API](#resource-api-new-in-react-19) *(new in 19)*
   - [`use`](#use)
8. [Other hooks](#other-hooks)
   - [`useId`](#useid)
   - [`useSyncExternalStore`](#usesyncexternalstore)
   - [`useDebugValue`](#usedebugvalue)
9. [Quick summary](#quick-summary)

---

## State hooks

### `useState`

Stores a value that, when changed, re-renders the component.

```tsx
import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState<number>(0);

  return (
    <button onClick={() => setCount((c) => c + 1)}>
      Clicked {count} times
    </button>
  );
}
```

### `useReducer`

Like `useState`, but the update logic lives in a single reducer function. Good for state with several related fields or many kinds of updates.

```tsx
import { useReducer } from "react";

type State = { count: number };
type Action = { type: "increment" } | { type: "decrement" } | { type: "reset" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment": return { count: state.count + 1 };
    case "decrement": return { count: state.count - 1 };
    case "reset":     return { count: 0 };
  }
}

export function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </>
  );
}
```

---

## Context hook

### `useContext`

Reads a value from the nearest matching context provider, so you don't have to pass props down through every level.

```tsx
import { createContext, useContext } from "react";

type Theme = "light" | "dark";
const ThemeContext = createContext<Theme>("light");

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={`btn-${theme}`}>I am {theme}</button>;
}

export function App() {
  // React 19: you can render <Context> directly instead of <Context.Provider>
  return (
    <ThemeContext value="dark">
      <ThemedButton />
    </ThemeContext>
  );
}
```

---

## Ref hooks

### `useRef`

Holds a mutable value that survives re-renders **without** causing one. Most often used to point at a DOM element.

```tsx
import { useRef } from "react";

export function FocusInput() {
  // React 19 types require an initial argument
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>Focus</button>
    </>
  );
}
```

### `useImperativeHandle`

Customises what a parent gets when it holds a ref to your component, e.g. exposing only a `focus()` method. In React 19, `ref` is a normal prop, so `forwardRef` is no longer needed.

```tsx
import { useImperativeHandle, useRef, type Ref } from "react";

export type FancyInputHandle = { focus: () => void; clear: () => void };

function FancyInput({ ref }: { ref?: Ref<FancyInputHandle> }) {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    clear: () => { if (inputRef.current) inputRef.current.value = ""; },
  }), []);

  return <input ref={inputRef} />;
}

export function Parent() {
  const fancyRef = useRef<FancyInputHandle>(null);
  return (
    <>
      <FancyInput ref={fancyRef} />
      <button onClick={() => fancyRef.current?.clear()}>Clear</button>
    </>
  );
}
```

---

## Effect hooks

### `useEffect`

Runs side effects (fetching, subscriptions, timers) **after** the component renders. Return a function to clean up.

```tsx
import { useEffect, useState } from "react";

export function Clock() {
  const [now, setNow] = useState<Date>(new Date());

  useEffect(() => {
    const id = setInterval(() => setNow(new Date()), 1000);
    return () => clearInterval(id); // cleanup
  }, []); // [] = run once on mount

  return <p>{now.toLocaleTimeString()}</p>;
}
```

### `useLayoutEffect`

Same as `useEffect`, but runs synchronously **before the browser paints**. Use it only when you must measure layout and adjust before the user sees anything (it blocks painting).

```tsx
import { useLayoutEffect, useRef, useState } from "react";

export function Tooltip({ text }: { text: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    if (ref.current) setHeight(ref.current.getBoundingClientRect().height);
  }, [text]);

  return (
    <div ref={ref} style={{ transform: `translateY(-${height}px)` }}>
      {text}
    </div>
  );
}
```

### `useInsertionEffect`

Fires before any layout effects, for injecting `<style>` tags. Intended for CSS-in-JS library authors — you'll rarely need it in app code.

```tsx
import { useInsertionEffect } from "react";

const injected = new Set<string>();

function useCss(rule: string): void {
  useInsertionEffect(() => {
    if (injected.has(rule)) return;
    injected.add(rule);
    const style = document.createElement("style");
    style.textContent = rule;
    document.head.appendChild(style);
  }, [rule]);
}

export function RedBox() {
  useCss(".red-box { background: red; padding: 8px; }");
  return <div className="red-box">Styled</div>;
}
```

### `useEffectEvent` *(React 19.2)*

Wraps a function so an effect can call it and always see the **latest** props/state, without that function being a dependency (so the effect doesn't re-run when those values change).

```tsx
import { useEffect, useEffectEvent } from "react";

declare function connect(roomId: string): { on(e: string, cb: () => void): void; close(): void };
declare function showToast(msg: string, theme: string): void;

export function ChatRoom({ roomId, theme }: { roomId: string; theme: string }) {
  // Always reads the current theme, but isn't a dependency
  const onConnected = useEffectEvent(() => {
    showToast("Connected!", theme);
  });

  useEffect(() => {
    const conn = connect(roomId);
    conn.on("connected", onConnected);
    return () => conn.close();
  }, [roomId]); // changing theme does NOT reconnect

  return <h1>Room: {roomId}</h1>;
}
```

---

## Performance hooks

> With the **React Compiler** enabled, much of the manual memoisation below (`useMemo`, `useCallback`) is done automatically.

### `useMemo`

Caches the result of an expensive calculation and only recomputes it when its dependencies change.

```tsx
import { useMemo } from "react";

type Product = { id: number; name: string; price: number };

export function ProductList({ products, query }: { products: Product[]; query: string }) {
  const filtered = useMemo<Product[]>(
    () => products.filter((p) => p.name.toLowerCase().includes(query.toLowerCase())),
    [products, query]
  );

  return <ul>{filtered.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

### `useCallback`

Caches a function definition so it keeps the same identity between renders. Useful when passing callbacks to memoised children.

```tsx
import { memo, useCallback, useState } from "react";

const SaveButton = memo(function SaveButton({ onSave }: { onSave: () => void }) {
  return <button onClick={onSave}>Save</button>;
});

export function Editor() {
  const [text, setText] = useState("");

  const handleSave = useCallback(() => {
    console.log("Saving", text);
  }, [text]);

  return (
    <>
      <textarea value={text} onChange={(e) => setText(e.target.value)} />
      <SaveButton onSave={handleSave} />
    </>
  );
}
```

### `useTransition`

Marks an update as non-urgent so the UI stays responsive while it happens. In React 19 the callback can be `async`.

```tsx
import { useState, useTransition } from "react";

declare function updateName(name: string): Promise<string | null>; // returns error or null

export function UpdateName() {
  const [name, setName] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const err = await updateName(name);
      setError(err);
    });
  };

  return (
    <>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        {isPending ? "Saving…" : "Update"}
      </button>
      {error && <p>{error}</p>}
    </>
  );
}
```

### `useDeferredValue`

Gives you a "lagging" copy of a value, so slow parts of the UI can update after the urgent parts. React 19 adds an optional initial value.

```tsx
import { useDeferredValue, useState } from "react";

declare function SlowResults(props: { query: string }): JSX.Element;

export function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query, ""); // "" on first render

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <div style={{ opacity: query !== deferredQuery ? 0.5 : 1 }}>
        <SlowResults query={deferredQuery} />
      </div>
    </>
  );
}
```

---

## Actions & forms hooks *(new in React 19)*

### `useActionState`

Manages state that is updated by a form action. Returns the current state, a wrapped action to pass to `<form action>`, and a pending flag.

```tsx
import { useActionState } from "react";

type FormState = { error: string | null; success: boolean };

declare function subscribe(email: string): Promise<void>;

async function subscribeAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const email = String(formData.get("email") ?? "");
  if (!email.includes("@")) return { error: "Invalid email", success: false };
  await subscribe(email);
  return { error: null, success: true };
}

export function Newsletter() {
  const [state, formAction, isPending] = useActionState(subscribeAction, {
    error: null,
    success: false,
  });

  return (
    <form action={formAction}>
      <input name="email" type="email" />
      <button disabled={isPending}>Subscribe</button>
      {state.error && <p>{state.error}</p>}
      {state.success && <p>Thanks!</p>}
    </form>
  );
}
```

### `useOptimistic`

Shows a temporary "optimistic" result immediately while an async action is in flight. It automatically reverts to the real state when the action finishes.

```tsx
import { useOptimistic } from "react";

type Message = { id: string; text: string; sending?: boolean };

export function Thread({
  messages,
  sendMessage,
}: {
  messages: Message[];
  sendMessage: (text: string) => Promise<void>;
}) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (current: Message[], text: string): Message[] => [
      ...current,
      { id: crypto.randomUUID(), text, sending: true },
    ]
  );

  async function formAction(formData: FormData) {
    const text = String(formData.get("text"));
    addOptimistic(text);
    await sendMessage(text);
  }

  return (
    <>
      {optimisticMessages.map((m) => (
        <p key={m.id}>{m.text} {m.sending && <small>(sending…)</small>}</p>
      ))}
      <form action={formAction}>
        <input name="text" />
        <button>Send</button>
      </form>
    </>
  );
}
```

### `useFormStatus` *(from `react-dom`)*

Lets a component **inside** a `<form>` read that form's submission status — no prop drilling needed. Perfect for reusable submit buttons.

```tsx
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending, data } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? `Saving ${data?.get("title") ?? ""}…` : "Save"}
    </button>
  );
}

export function PostForm({ action }: { action: (fd: FormData) => Promise<void> }) {
  return (
    <form action={action}>
      <input name="title" />
      <SubmitButton />
    </form>
  );
}
```

---

## Resource API *(new in React 19)*

### `use`

Reads the value of a **Promise** (suspending until it resolves) or a **Context**. Unlike other hooks, it can be called inside conditions and loops.

```tsx
import { Suspense, use, createContext } from "react";

type User = { id: number; name: string };

// Reading a promise (create it outside render, e.g. in a parent or loader)
function UserName({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // suspends until resolved
  return <h2>{user.name}</h2>;
}

export function Profile({ userPromise }: { userPromise: Promise<User> }) {
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <UserName userPromise={userPromise} />
    </Suspense>
  );
}

// Reading context conditionally
const ThemeContext = createContext<"light" | "dark">("light");

function Heading({ show }: { show: boolean }) {
  if (!show) return null;
  const theme = use(ThemeContext); // allowed after an early return
  return <h1 className={theme}>Hello</h1>;
}
```

---

## Other hooks

### `useId`

Generates a unique, stable ID that matches between server and client. Use it for accessibility attributes like `htmlFor` / `aria-describedby` — not for list keys.

```tsx
import { useId } from "react";

export function EmailField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" aria-describedby={`${id}-hint`} />
      <p id={`${id}-hint`}>We'll never share it.</p>
    </>
  );
}
```

### `useSyncExternalStore`

Subscribes to a store that lives outside React (browser APIs, a global store) safely with concurrent rendering.

```tsx
import { useSyncExternalStore } from "react";

function subscribe(callback: () => void): () => void {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);
  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

export function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine, // client snapshot
    () => true              // server snapshot
  );
}

export function StatusBar() {
  const isOnline = useOnlineStatus();
  return <p>{isOnline ? "✅ Online" : "❌ Offline"}</p>;
}
```

### `useDebugValue`

Adds a label to a **custom hook** in React DevTools. Has no effect on behaviour.

```tsx
import { useDebugValue, useSyncExternalStore } from "react";

declare function subscribe(cb: () => void): () => void;

export function useOnlineStatus(): boolean {
  const isOnline = useSyncExternalStore(subscribe, () => navigator.onLine);
  useDebugValue(isOnline ? "Online" : "Offline");
  return isOnline;
}
```

---

## Quick summary

| Hook | One-liner |
|---|---|
| `useState` | Local state that triggers re-renders |
| `useReducer` | State managed by a reducer function |
| `useContext` | Read a value from context |
| `useRef` | Mutable value / DOM reference, no re-render |
| `useImperativeHandle` | Customise what a ref exposes to parents |
| `useEffect` | Side effects after render |
| `useLayoutEffect` | Side effects before paint (layout measuring) |
| `useInsertionEffect` | Inject styles before layout effects (libraries) |
| `useEffectEvent` | Non-reactive event logic called from effects (19.2) |
| `useMemo` | Cache an expensive computed value |
| `useCallback` | Cache a function's identity |
| `useTransition` | Mark updates as non-urgent; supports async |
| `useDeferredValue` | Lagging copy of a value for slow UI |
| `useActionState` | State driven by form actions (19) |
| `useOptimistic` | Instant optimistic UI during async work (19) |
| `useFormStatus` | Read parent form's pending status (19, react-dom) |
| `use` | Read a Promise or Context, even conditionally (19) |
| `useId` | Stable unique IDs for accessibility |
| `useSyncExternalStore` | Subscribe to external stores safely |
| `useDebugValue` | Label custom hooks in DevTools |
