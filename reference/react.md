# React Code Review Guide

Focus areas: Rules of Hooks, measured performance tuning, component design, and modern React 19 / RSC patterns.

## Contents

- [Rules of Hooks](#rules-of-hooks)
- [useEffect patterns](#useeffect-patterns)
- [useMemo / useCallback](#usememo--usecallback)
- [Component design](#component-design)
- [Error Boundaries & Suspense](#error-boundaries--suspense)
- [Server Components (RSC)](#server-components-rsc)
- [React 19 Actions & Forms](#react-19-actions--forms)
- [Suspense & streaming SSR](#suspense--streaming-ssr)
- [TanStack Query v5](#tanstack-query-v5)
- [Review checklists](#review-checklists)

---

## Rules of Hooks

```tsx
// ❌ Conditional hooks — violates Rules of Hooks
function BadComponent({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null);  // Error!
  }
  return <div>...</div>;
}

// ✅ Hooks only at the top level of the component
function GoodComponent({ isLoggedIn }) {
  const [user, setUser] = useState(null);
  if (!isLoggedIn) return <LoginPrompt />;
  return <div>{user?.name}</div>;
}
```

---

## useEffect patterns

```tsx
// ❌ Missing or incomplete dependency array
function BadEffect({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, []);  // missing userId!
}

// ✅ Complete deps + cleanup
function GoodEffect({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(data => {
      if (!cancelled) setUser(data);
    });
    return () => { cancelled = true; };
  }, [userId]);
}

// ❌ useEffect for derived state (anti-pattern)
function BadDerived({ items }) {
  const [filteredItems, setFilteredItems] = useState([]);
  useEffect(() => {
    setFilteredItems(items.filter(i => i.active));
  }, [items]);  // extra effect + extra render
  return <List items={filteredItems} />;
}

// ✅ Derive during render or useMemo
function GoodDerived({ items }) {
  const filteredItems = useMemo(
    () => items.filter(i => i.active),
    [items]
  );
  return <List items={filteredItems} />;
}

// ❌ useEffect for event-driven side effects
function BadEventEffect() {
  const [query, setQuery] = useState('');
  useEffect(() => {
    if (query) {
      analytics.track('search', { query });  // belongs in the event handler
    }
  }, [query]);
}

// ✅ Run side effects in the handler
function GoodEvent() {
  const [query, setQuery] = useState('');
  const handleSearch = (q: string) => {
    setQuery(q);
    analytics.track('search', { query: q });
  };
}
```

---

## useMemo / useCallback

```tsx
// ❌ Premature optimization — constants don’t need useMemo
function OverOptimized() {
  const config = useMemo(() => ({ timeout: 5000 }), []);  // pointless
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);  // pointless if not passed to a memoized child
}

// ✅ Optimize only when measured / justified
function ProperlyOptimized() {
  const config = { timeout: 5000 };
  const handleClick = () => console.log('clicked');
}

// ❌ useCallback deps always change
function BadCallback({ data }) {
  // new data reference every render → callback never stable
  const process = useCallback(() => {
    return data.map(transform);
  }, [data]);
}

// ✅ useMemo + useCallback with React.memo
const MemoizedChild = React.memo(function Child({ onClick, items }) {
  return <div onClick={onClick}>{items.length}</div>;
});

function Parent({ rawItems }) {
  const items = useMemo(() => processItems(rawItems), [rawItems]);
  const handleClick = useCallback(() => {
    console.log(items.length);
  }, [items]);
  return <MemoizedChild onClick={handleClick} items={items} />;
}
```

---

## Component design

```tsx
// ❌ Component defined inside another — new type every render
function BadParent() {
  function ChildComponent() {  // new function each render
    return <div>child</div>;
  }
  return <ChildComponent />;
}

// ✅ Define at module scope
function ChildComponent() {
  return <div>child</div>;
}
function GoodParent() {
  return <ChildComponent />;
}

// ❌ Props are new object/function every render
function BadProps() {
  return (
    <MemoizedComponent
      style={{ color: 'red' }}  // new object
      onClick={() => {}}         // new function
    />
  );
}

// ✅ Stable references where it matters
const style = { color: 'red' };
function GoodProps() {
  const handleClick = useCallback(() => {}, []);
  return <MemoizedComponent style={style} onClick={handleClick} />;
}
```

---

## Error Boundaries & Suspense

```tsx
// ❌ No error boundary
function BadApp() {
  return (
    <Suspense fallback={<Loading />}>
      <DataComponent />  {/* uncaught error can crash the tree */}
    </Suspense>
  );
}

// ✅ Error boundary wraps Suspense
function GoodApp() {
  return (
    <ErrorBoundary fallback={<ErrorUI />}>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## Server Components (RSC)

```tsx
// ❌ Client-only APIs in a Server Component
// app/page.tsx (Server Component by default)
function BadServerComponent() {
  const [count, setCount] = useState(0);  // Error! No hooks in RSC
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// ✅ Move interactivity to a Client Component
// app/counter.tsx
'use client';
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// app/page.tsx (Server Component)
async function GoodServerComponent() {
  const data = await fetchData();  // can await directly
  return (
    <div>
      <h1>{data.title}</h1>
      <Counter />
    </div>
  );
}

// ❌ Misplaced 'use client' — clientifies the whole subtree
// layout.tsx
'use client';  // every descendant becomes a client component
export default function Layout({ children }) { ... }

// ✅ Use 'use client' only on components that need it
// Keep client boundaries at leaves when possible
```

---

## React 19 Actions & Forms

React 19 adds Actions and new form hooks for async work and optimistic UI.

### useActionState

```tsx
// ❌ Old way: many pieces of state
function OldForm() {
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [data, setData] = useState(null);

  const handleSubmit = async (formData: FormData) => {
    setIsPending(true);
    setError(null);
    try {
      const result = await submitForm(formData);
      setData(result);
    } catch (e) {
      setError(e.message);
    } finally {
      setIsPending(false);
    }
  };
}

// ✅ React 19: useActionState bundles pending + result
import { useActionState } from 'react';

function NewForm() {
  const [state, formAction, isPending] = useActionState(
    async (prevState, formData: FormData) => {
      try {
        const result = await submitForm(formData);
        return { success: true, data: result };
      } catch (e) {
        return { success: false, error: e.message };
      }
    },
    { success: false, data: null, error: null }
  );

  return (
    <form action={formAction}>
      <input name="email" />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### useFormStatus

```tsx
// ❌ Threading submitting state through props
function BadSubmitButton({ isSubmitting }) {
  return <button disabled={isSubmitting}>Submit</button>;
}

// ✅ useFormStatus reads parent <form> state (no prop drilling)
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  // Must run in a descendant of <form>
  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

// ❌ useFormStatus next to the form — not a descendant, won’t work
function BadForm() {
  const { pending } = useFormStatus();  // wrong place
  return (
    <form action={action}>
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ useFormStatus inside a child of the form
function GoodForm() {
  return (
    <form action={action}>
      <SubmitButton />
    </form>
  );
}
```

### useOptimistic

```tsx
// ❌ UI waits for the server
function SlowLike({ postId, likes }) {
  const [likeCount, setLikeCount] = useState(likes);
  const [isPending, setIsPending] = useState(false);

  const handleLike = async () => {
    setIsPending(true);
    const newCount = await likePost(postId);
    setLikeCount(newCount);
    setIsPending(false);
  };
}

// ✅ useOptimistic — instant UI, auto rollback on failure
import { useOptimistic } from 'react';

function FastLike({ postId, likes }) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (currentLikes, increment: number) => currentLikes + increment
  );

  const handleLike = async () => {
    addOptimisticLike(1);
    try {
      await likePost(postId);
    } catch {
      // React rolls back to `likes`
    }
  };

  return <button onClick={handleLike}>{optimisticLikes} likes</button>;
}
```

### Server Actions (Next.js 15+)

```tsx
// ❌ Client-only fetch to an API route
'use client';
function ClientForm() {
  const handleSubmit = async (formData: FormData) => {
    const res = await fetch('/api/submit', {
      method: 'POST',
      body: formData,
    });
    // ...
  };
}

// ✅ Server Action + useActionState
// actions.ts
'use server';
export async function createPost(prevState: any, formData: FormData) {
  const title = formData.get('title');
  await db.posts.create({ title });
  revalidatePath('/posts');
  return { success: true };
}

// form.tsx
'use client';
import { createPost } from './actions';

function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null);
  return (
    <form action={formAction}>
      <input name="title" />
      <SubmitButton />
    </form>
  );
}
```

---

## Suspense & streaming SSR

Suspense and streaming are core in React 18+ and common in Next.js 15+ (2025).

### Basic Suspense

```tsx
// ❌ Manual loading flags
function OldComponent() {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetchData().then(setData).finally(() => setIsLoading(false));
  }, []);

  if (isLoading) return <Spinner />;
  return <DataView data={data} />;
}

// ✅ Declarative loading with Suspense
function NewComponent() {
  return (
    <Suspense fallback={<Spinner />}>
      <DataView />  {/* uses use() or a Suspense-aware data layer */}
    </Suspense>
  );
}
```

### Multiple Suspense boundaries

```tsx
// ❌ Single boundary — everything waits on the slowest child
function BadLayout() {
  return (
    <Suspense fallback={<FullPageSpinner />}>
      <Header />
      <MainContent />  {/* slow */}
      <Sidebar />      {/* fast */}
    </Suspense>
  );
}

// ✅ Split boundaries — stream independently
function GoodLayout() {
  return (
    <>
      <Header />
      <div className="flex">
        <Suspense fallback={<ContentSkeleton />}>
          <MainContent />
        </Suspense>
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>
      </div>
    </>
  );
}
```

### Next.js 15 streaming

```tsx
// app/page.tsx — automatic streaming
export default async function Page() {
  // This await doesn’t have to block the whole shell (with streaming)
  const data = await fetchSlowData();
  return <div>{data}</div>;
}

// app/loading.tsx — route-level Suspense fallback
export default function Loading() {
  return <Skeleton />;
}
```

### `use()` hook (React 19)

```tsx
// ✅ Read a Promise from props/context
import { use } from 'react';

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);  // suspends the boundary
  return (
    <ul>
      {comments.map(c => <li key={c.id}>{c.text}</li>)}
    </ul>
  );
}

// Parent creates the Promise; child consumes
function Post({ postId }) {
  const commentsPromise = fetchComments(postId);  // don’t await here
  return (
    <article>
      <PostContent id={postId} />
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </article>
  );
}
```

---

## TanStack Query v5

TanStack Query is the dominant data-fetching layer for React; v5 is the current stable line.

### Default client options

```tsx
// ❌ Default QueryClient — often too chatty
const queryClient = new QueryClient();

// ✅ Sensible production defaults
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // fresh for 5 minutes
      gcTime: 1000 * 60 * 30,    // cache GC (renamed from cacheTime in v5)
      retry: 3,
      refetchOnWindowFocus: false,  // tune to product needs
    },
  },
});
```

### `queryOptions` (v5)

```tsx
// ❌ Duplicated queryKey + queryFn
function Component1() {
  const { data } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });
}

function prefetchUser(queryClient, userId) {
  queryClient.prefetchQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });
}

// ✅ queryOptions — single definition, typed keys
import { queryOptions } from '@tanstack/react-query';

const userQueryOptions = (userId: string) =>
  queryOptions({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });

function Component1({ userId }) {
  const { data } = useQuery(userQueryOptions(userId));
}

function prefetchUser(queryClient, userId) {
  queryClient.prefetchQuery(userQueryOptions(userId));
}

const user = queryClient.getQueryData(userQueryOptions(userId).queryKey);
```

### Common pitfalls

```tsx
// ❌ staleTime 0 → refetch on every mount/focus
useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
});

// ✅ Tune staleTime
useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  staleTime: 1000 * 60,
});

// ❌ queryKey omits inputs used in queryFn
function BadQuery({ filters }) {
  useQuery({
    queryKey: ['items'],
    queryFn: () => fetchItems(filters),
  });
}

// ✅ queryKey includes every input that affects the result
function GoodQuery({ filters }) {
  useQuery({
    queryKey: ['items', filters],
    queryFn: () => fetchItems(filters),
  });
}
```

### useSuspenseQuery

> **Important:** `useSuspenseQuery` behaves differently from `useQuery`; pick deliberately.

#### Comparison

| Feature | useQuery | useSuspenseQuery |
|---------|----------|------------------|
| `enabled` | ✅ | ❌ |
| `placeholderData` | ✅ | ❌ |
| `data` type | `T \| undefined` | `T` (defined when render succeeds) |
| Errors | `error` field | Propagates to Error Boundary |
| Loading | `isLoading`, etc. | Suspends |

#### No `enabled` — use composition

```tsx
// ❌ enabled on useSuspenseQuery
function BadSuspenseQuery({ userId }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    enabled: !!userId,  // not supported
  });
}

// ✅ Guard in the parent; child always has userId
function GoodSuspenseQuery({ userId }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return <UserProfile user={data} />;
}

function Parent({ userId }) {
  if (!userId) return <NoUserSelected />;
  return (
    <Suspense fallback={<UserSkeleton />}>
      <GoodSuspenseQuery userId={userId} />
    </Suspense>
  );
}
```

#### Errors

```tsx
// ❌ useSuspenseQuery has no meaningful `error` on success path
function BadErrorHandling() {
  const { data, error } = useSuspenseQuery({...});
  if (error) return <Error />;  // not how suspense errors work
}

// ✅ Error Boundary + Suspense
function GoodErrorHandling() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}

function DataComponent() {
  const { data } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: fetchData,
  });
  return <Display data={data} />;
}
```

#### When to use `useSuspenseQuery`

```tsx
// ✅ Good fits:
// 1. Data is always required (no “skip” query)
// 2. UI shouldn’t render without data
// 3. Suspense-first app / React 19 patterns
// 4. RSC + client hydration patterns

// ❌ Poor fits:
// 1. Truly conditional fetches
// 2. Need placeholderData / seeded cache
// 3. Inline loading/error UI in the same component
// 4. Chained dependent queries (use careful composition)

// ✅ Parallel suspense queries
function MultipleQueries({ userId }) {
  const [userQuery, postsQuery] = useSuspenseQueries({
    queries: [
      { queryKey: ['user', userId], queryFn: () => fetchUser(userId) },
      { queryKey: ['posts', userId], queryFn: () => fetchPosts(userId) },
    ],
  });
  return <Profile user={userQuery.data} posts={postsQuery.data} />;
}
```

### Optimistic updates (v5)

```tsx
// ❌ Manual cache rollback (verbose)
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });
    const previousTodos = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});

// ✅ v5 pattern: show pending row via mutation variables
function TodoList() {
  const { data: todos } = useQuery(todosQueryOptions);
  const { mutate, variables, isPending } = useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <ul>
      {todos?.map(todo => <TodoItem key={todo.id} todo={todo} />)}
      {isPending && <TodoItem todo={variables} isOptimistic />}
    </ul>
  );
}
```

### v5 loading flags

```tsx
// v4: isLoading covered more cases
// v5: isPending = no cached data yet; isLoading = isPending && isFetching

const { data, isPending, isFetching, isLoading } = useQuery({...});

// isPending: no data in cache yet
// isFetching: request in flight (including background)
// isLoading: first load in progress

// ❌ Blind v4 → v5 migration
if (isLoading) return <Spinner />;  // semantics shifted

// ✅ Name the UX you want
if (isPending) return <Spinner />;
// or
if (isLoading) return <Spinner />;
```

---

## Review checklists

### Rules of Hooks

- [ ] Hooks only at top level of components / custom hooks
- [ ] No hooks in conditions or loops
- [ ] `useEffect` dependency arrays complete
- [ ] Cleanup for subscriptions, timers, in-flight requests
- [ ] No `useEffect` for pure derivation

### Performance (only when justified)

- [ ] `useMemo` / `useCallback` where they actually help
- [ ] `React.memo` paired with stable props
- [ ] No inner component definitions
- [ ] No fresh object/function props to memoized children unless intentional
- [ ] Virtualize very long lists (`react-window` / `react-virtual`)

### Component design

- [ ] Single responsibility; avoid 200+ line components
- [ ] Logic vs presentation split (custom hooks)
- [ ] Clear props; TypeScript for public APIs
- [ ] Avoid prop drilling (Context or composition)

### State

- [ ] State colocated (smallest scope that works)
- [ ] `useReducer` for complex local state
- [ ] Global state via Context or a store
- [ ] Prefer derived state over duplicated state

### Errors

- [ ] Error boundaries around risky UI
- [ ] Suspense paired with boundaries where errors can throw
- [ ] Async paths handle failures

### Server Components

- [ ] `'use client'` only where the browser is required
- [ ] No hooks or DOM events in Server Components
- [ ] Prefer client boundaries at leaves
- [ ] Fetch on the server when possible

### React 19 forms

- [ ] `useActionState` instead of many `useState`s for forms
- [ ] `useFormStatus` inside a `<form>` descendant
- [ ] No `useOptimistic` for money-critical flows
- [ ] Server Actions marked `'use server'`

### Suspense & streaming

- [ ] Boundaries match UX (don’t block fast UI on slow data)
- [ ] Boundaries paired with error handling where needed
- [ ] Meaningful fallbacks (skeletons > generic spinners)
- [ ] Avoid slow `await` in shared layouts without streaming

### TanStack Query

- [ ] `queryKey` includes every query input
- [ ] Sensible `staleTime` (not always 0)
- [ ] No `enabled` with `useSuspenseQuery`
- [ ] Mutations invalidate or update cache
- [ ] Understand `isPending` vs `isLoading` / `isFetching`

### Testing

- [ ] `@testing-library/react`
- [ ] Prefer `screen` queries
- [ ] `userEvent` over `fireEvent`
- [ ] Prefer `*ByRole`
- [ ] Assert behavior, not implementation details
