---
name: zustand-5
description: Implements client-side React state with Zustand 5: stores with TypeScript, selectors and useShallow, persist and immer middleware, slices pattern, async actions, DevTools. Use when managing React state with Zustand, creating or refactoring stores, or when the user mentions zustand, global state, or store.
---

# Zustand 5

## Basic Store

Define an interface for state and actions; use `create<Store>(set => ...)`. Prefer functional updates `set((state) => ({ ... }))` for correctness.

```typescript
import { create } from "zustand";

interface CounterStore {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

function Counter() {
  const { count, increment, decrement } = useCounterStore();
  return (
    <div>
      <span>{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}
```

## Persist Middleware

Use `create<Store>()(persist(...))` for localStorage. TypeScript requires the double invocation `create<T>()(...)` when using middleware.

```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface SettingsStore {
  theme: "light" | "dark";
  language: string;
  setTheme: (theme: "light" | "dark") => void;
  setLanguage: (language: string) => void;
}

const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      theme: "light",
      language: "en",
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    { name: "settings-storage" }
  )
);
```

## Selectors

Select specific fields to avoid re-renders on unrelated state changes. Use `useShallow` when selecting multiple fields.

```typescript
// ✅ Single field
function UserName() {
  const name = useUserStore((state) => state.name);
  return <span>{name}</span>;
}

// ✅ Multiple fields
import { useShallow } from "zustand/react/shallow";

function UserInfo() {
  const { name, email } = useUserStore(
    useShallow((state) => ({ name: state.name, email: state.email }))
  );
  return <div>{name} - {email}</div>;
}

// ❌ AVOID: entire store re-renders on any change
const store = useUserStore();
```

## Async Actions

Set `loading` and `error` in `set` before and after the async call.

```typescript
interface UserStore {
  user: User | null;
  loading: boolean;
  error: string | null;
  fetchUser: (id: string) => Promise<void>;
}

const useUserStore = create<UserStore>((set) => ({
  user: null,
  loading: false,
  error: null,

  fetchUser: async (id) => {
    set({ loading: true, error: null });
    try {
      const response = await fetch(`/api/users/${id}`);
      const user = await response.json();
      set({ user, loading: false });
    } catch {
      set({ error: "Failed to fetch user", loading: false });
    }
  },
}));
```

## Slices Pattern

Split store by domain into slice factories; combine with spread in a single `create` call.

```typescript
// userSlice.ts
interface UserSlice {
  user: User | null;
  setUser: (user: User) => void;
  clearUser: () => void;
}

const createUserSlice = (set): UserSlice => ({
  user: null,
  setUser: (user) => set({ user }),
  clearUser: () => set({ user: null }),
});

// cartSlice.ts
interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
}

const createCartSlice = (set): CartSlice => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter((i) => i.id !== id),
  })),
});

// store.ts
type Store = UserSlice & CartSlice;

const useStore = create<Store>()((...args) => ({
  ...createUserSlice(...args),
  ...createCartSlice(...args),
}));
```

## Immer Middleware

Use `immer` to mutate draft state inside `set` for nested updates.

```typescript
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

interface TodoStore {
  todos: Todo[];
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
}

const useTodoStore = create<TodoStore>()(
  immer((set) => ({
    todos: [],

    addTodo: (text) => set((state) => {
      state.todos.push({ id: crypto.randomUUID(), text, done: false });
    }),

    toggleTodo: (id) => set((state) => {
      const todo = state.todos.find((t) => t.id === id);
      if (todo) todo.done = !todo.done;
    }),
  }))
);
```

## DevTools

Wrap the store with `devtools` for Redux DevTools inspection.

```typescript
import { create } from "zustand";
import { devtools } from "zustand/middleware";

const useStore = create<Store>()(
  devtools(
    (set) => ({ /* store definition */ }),
    { name: "MyStore" }
  )
);
```

## Outside React

Use `getState()` for one-off access and `subscribe()` for reactions outside components.

```typescript
const { count, increment } = useCounterStore.getState();
increment();

const unsubscribe = useCounterStore.subscribe(
  (state) => console.log("Count changed:", state.count)
);
```
