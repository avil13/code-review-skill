# Common Bugs Checklist

Language-specific bugs and issues to watch for during code review.

## Universal Issues

### Logic Errors
- [ ] Off-by-one errors in loops and array access
- [ ] Incorrect boolean logic (De Morgan's law violations)
- [ ] Missing null/undefined checks
- [ ] Race conditions in concurrent code
- [ ] Incorrect comparison operators (== vs ===, = vs ==)
- [ ] Integer overflow/underflow
- [ ] Floating point comparison issues

### Resource Management
- [ ] Memory leaks (unclosed connections, listeners)
- [ ] File handles not closed
- [ ] Database connections not released
- [ ] Event listeners not removed
- [ ] Timers/intervals not cleared

### Error Handling
- [ ] Swallowed exceptions (empty catch blocks)
- [ ] Generic exception handling hiding specific errors
- [ ] Missing error propagation
- [ ] Incorrect error types thrown
- [ ] Missing finally/cleanup blocks

## TypeScript/JavaScript

### Type Issues
```typescript
// ❌ Using any defeats type safety
function process(data: any) { return data.value; }

// ✅ Use proper types
interface Data { value: string; }
function process(data: Data) { return data.value; }
```

### Async/Await Pitfalls
```typescript
// ❌ Missing await
async function fetch() {
  const data = fetchData();  // Missing await!
  return data.json();
}

// ❌ Unhandled promise rejection
async function risky() {
  const result = await fetchData();  // No try-catch
  return result;
}

// ✅ Proper error handling
async function safe() {
  try {
    const result = await fetchData();
    return result;
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

### React Specific

#### Hooks rule violations
```tsx
// ❌ Conditional hooks — violates Rules of Hooks
function BadComponent({ show }) {
  if (show) {
    const [value, setValue] = useState(0);  // Error!
  }
  return <div>...</div>;
}

// ✅ Hooks must run unconditionally at the top level
function GoodComponent({ show }) {
  const [value, setValue] = useState(0);
  if (!show) return null;
  return <div>{value}</div>;
}

// ❌ Hooks inside a loop
function BadLoop({ items }) {
  items.forEach(item => {
    const [selected, setSelected] = useState(false);  // Error!
  });
}

// ✅ Lift state or use a different data structure
function GoodLoop({ items }) {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  return items.map(item => (
    <Item key={item.id} selected={selectedIds.has(item.id)} />
  ));
}
```

#### Common `useEffect` mistakes
```tsx
// ❌ Incomplete dependency array — stale closure
function StaleClosureExample({ userId, onSuccess }) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetchData(userId).then(result => {
      setData(result);
      onSuccess(result);  // onSuccess may be stale!
    });
  }, [userId]);  // missing onSuccess
}

// ✅ Complete dependency array
useEffect(() => {
  fetchData(userId).then(result => {
    setData(result);
    onSuccess(result);
  });
}, [userId, onSuccess]);

// ❌ Infinite loop — effect updates its own dependency
function InfiniteLoop() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setCount(count + 1);  // re-render retriggers effect
  }, [count]);  // infinite loop!
}

// ❌ Missing cleanup — memory leak
function MemoryLeak({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(userId).then(setUser);  // setState after unmount
  }, [userId]);
}

// ✅ Proper cleanup
function NoLeak({ userId }) {
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
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((a, b) => a + b.price, 0));
  }, [items]);  // unnecessary effect + extra render
}

// ✅ Derive directly or use useMemo
function GoodDerived({ items }) {
  const total = useMemo(
    () => items.reduce((a, b) => a + b.price, 0),
    [items]
  );
}

// ❌ useEffect for event-driven work
function BadEvent() {
  const [query, setQuery] = useState('');
  useEffect(() => {
    if (query) logSearch(query);  // belongs in an event handler
  }, [query]);
}

// ✅ Side effects in event handlers
function GoodEvent() {
  const handleSearch = (q: string) => {
    setQuery(q);
    logSearch(q);
  };
}
```

#### Misusing `useMemo` / `useCallback`
```tsx
// ❌ Premature optimization — constants don’t need memo
function OverOptimized() {
  const config = useMemo(() => ({ api: '/v1' }), []);  // pointless
  const noop = useCallback(() => {}, []);  // pointless
}

// ❌ useMemo with empty deps (can hide bugs)
function EmptyDeps({ user }) {
  const greeting = useMemo(() => `Hello ${user.name}`, []);
  // greeting won’t update when user changes!
}

// ❌ useCallback deps always change
function UselessCallback({ data }) {
  const process = useCallback(() => {
    return data.map(transform);
  }, [data]);  // useless if data is a new reference every render
}

// ❌ useMemo/useCallback without React.memo on the child
function Parent() {
  const data = useMemo(() => compute(), []);
  const handler = useCallback(() => {}, []);
  return <Child data={data} onClick={handler} />;
  // Child isn’t memoized — optimizations have no effect
}

// ✅ Pair memoization correctly
const MemoChild = React.memo(function Child({ data, onClick }) {
  return <button onClick={onClick}>{data}</button>;
});

function Parent() {
  const data = useMemo(() => expensiveCompute(), [dep]);
  const handler = useCallback(() => {}, []);
  return <MemoChild data={data} onClick={handler} />;
}
```

#### Component design issues
```tsx
// ❌ Defining a component inside another
function Parent() {
  // New Child function each render → full remount
  const Child = () => <div>child</div>;
  return <Child />;
}

// ✅ Define components at module scope
const Child = () => <div>child</div>;
function Parent() {
  return <Child />;
}

// ❌ Props are always new references — breaks memo
function BadProps() {
  return (
    <MemoComponent
      style={{ color: 'red' }}      // new object each render
      onClick={() => handle()}       // new function each render
      items={data.filter(x => x)}    // new array each render
    />
  );
}

// ❌ Mutating props
function MutateProps({ user }) {
  user.name = 'Changed';  // never do this
  return <div>{user.name}</div>;
}
```

#### Server Components mistakes (React 19+)
```tsx
// ❌ Client APIs inside a Server Component
// app/page.tsx (Server Component by default)
export default function Page() {
  const [count, setCount] = useState(0);  // Error!
  useEffect(() => {}, []);  // Error!
  return <button onClick={() => {}}>Click</button>;  // Error!
}

// ✅ Move interactivity to a Client Component
// app/counter.tsx
'use client';
export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// app/page.tsx
import { Counter } from './counter';
export default async function Page() {
  const data = await fetchData();  // Server Components can await directly
  return <Counter initialCount={data.count} />;
}

// ❌ 'use client' on a parent — entire subtree becomes client-only
// layout.tsx
'use client';  // bad idea — every descendant is a client component
export default function Layout({ children }) { ... }
```

#### Common testing mistakes
```tsx
// ❌ Querying via container
const { container } = render(<Component />);
const button = container.querySelector('button');  // discouraged

// ✅ Use screen and semantic queries
render(<Component />);
const button = screen.getByRole('button', { name: /submit/i });

// ❌ Using fireEvent
fireEvent.click(button);

// ✅ Prefer userEvent
await userEvent.click(button);

// ❌ Testing implementation details
expect(component.state.isOpen).toBe(true);

// ✅ Test behavior
expect(screen.getByRole('dialog')).toBeVisible();

// ❌ Awaiting a synchronous query
await screen.getByText('Hello');  // getBy is synchronous

// ✅ Use findBy for async updates
await screen.findByText('Hello');  // findBy waits
```

### React Common Mistakes Checklist
- [ ] Hooks not at top level (inside conditions/loops)
- [ ] Incomplete `useEffect` dependency arrays
- [ ] `useEffect` without cleanup when needed
- [ ] `useEffect` used to compute derived state
- [ ] Overuse of `useMemo` / `useCallback`
- [ ] `useMemo` / `useCallback` without `React.memo` on children
- [ ] Child components defined inside parents
- [ ] New object/function props on every render (when child is memoized)
- [ ] Mutating props directly
- [ ] Missing list keys or using index as key
- [ ] Client hooks/APIs in Server Components
- [ ] `'use client'` on a parent that clientifies the whole tree
- [ ] Queries via `container` instead of `screen`
- [ ] Testing implementation instead of behavior

### React 19 Actions & Forms mistakes

```tsx
// === useActionState mistakes ===

// ❌ Calling setState in the action instead of returning state
const [state, action] = useActionState(async (prev, formData) => {
  setSomeState(newValue);  // wrong — return new state
}, initialState);

// ✅ Return new state
const [state, action] = useActionState(async (prev, formData) => {
  const result = await submitForm(formData);
  return { ...prev, data: result };
}, initialState);

// ❌ Ignoring isPending
const [state, action] = useActionState(submitAction, null);
return <button>Submit</button>;  // user can double-submit

// ✅ Disable submit while pending
const [state, action, isPending] = useActionState(submitAction, null);
return <button disabled={isPending}>Submit</button>;

// === useFormStatus mistakes ===

// ❌ useFormStatus next to the form (sibling, not descendant)
function Form() {
  const { pending } = useFormStatus();  // always undefined!
  return <form><button disabled={pending}>Submit</button></form>;
}

// ✅ Call from a child of the form
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}
function Form() {
  return <form><SubmitButton /></form>;
}

// === useOptimistic mistakes ===

// ❌ For critical flows (payments, etc.)
function PaymentButton() {
  const [optimisticPaid, setPaid] = useOptimistic(false);
  const handlePay = async () => {
    setPaid(true);  // dangerous: shows paid before success
    await processPayment();
  };
}

// ❌ No UX after rollback
const [optimisticLikes, addLike] = useOptimistic(likes);
// On failure UI reverts — user may not understand why the like vanished

// ✅ Surface failures
const handleLike = async () => {
  addLike(1);
  try {
    await likePost();
  } catch {
    toast.error('Could not like post. Try again.');
  }
};
```

### React 19 Forms Checklist
- [ ] `useActionState` returns new state, not ad-hoc `setState`
- [ ] `useActionState` uses `isPending` to guard submit
- [ ] `useFormStatus` used inside a form descendant
- [ ] `useOptimistic` not used for money-critical actions
- [ ] User feedback when optimistic updates fail
- [ ] Server Actions marked with `'use server'`

### Suspense & Streaming mistakes

```tsx
// === Suspense boundary mistakes ===

// ❌ One Suspense around the whole page — slow UI blocks fast UI
function BadPage() {
  return (
    <Suspense fallback={<FullPageLoader />}>
      <FastHeader />      {/* fast */}
      <SlowMainContent /> {/* slow — blocks everything */}
      <FastFooter />      {/* fast */}
    </Suspense>
  );
}

// ✅ Split boundaries so slow regions don’t block the rest
function GoodPage() {
  return (
    <>
      <FastHeader />
      <Suspense fallback={<ContentSkeleton />}>
        <SlowMainContent />
      </Suspense>
      <FastFooter />
    </>
  );
}

// ❌ No Error Boundary
function NoErrorHandling() {
  return (
    <Suspense fallback={<Loading />}>
      <DataFetcher />  {/* error → blank screen */}
    </Suspense>
  );
}

// ✅ Error Boundary + Suspense
function WithErrorHandling() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Suspense fallback={<Loading />}>
        <DataFetcher />
      </Suspense>
    </ErrorBoundary>
  );
}

// === use() hook mistakes ===

// ❌ Creating the Promise during render (new Promise each render)
function BadUse() {
  const data = use(fetchData());  // new Promise every render!
  return <div>{data}</div>;
}

// ✅ Create in parent and pass as prop
function Parent() {
  const dataPromise = useMemo(() => fetchData(), []);
  return <Child dataPromise={dataPromise} />;
}
function Child({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}

// === Next.js streaming mistakes ===

// ❌ Awaiting slow data in layout.tsx — blocks all child routes
// app/layout.tsx
export default async function Layout({ children }) {
  const config = await fetchSlowConfig();  // blocks the whole app shell
  return <ConfigProvider value={config}>{children}</ConfigProvider>;
}

// ✅ Move slow fetches to pages or wrap with Suspense
// app/layout.tsx
export default function Layout({ children }) {
  return (
    <Suspense fallback={<ConfigSkeleton />}>
      <ConfigProvider>{children}</ConfigProvider>
    </Suspense>
  );
}
```

### Suspense Checklist
- [ ] Slow regions have their own Suspense boundary
- [ ] Each Suspense has a matching Error Boundary where needed
- [ ] Fallbacks are meaningful skeletons (not only a spinner)
- [ ] Promises passed to `use()` are not created during render
- [ ] No slow `await` in shared layouts without streaming
- [ ] Nesting depth stays reasonable (e.g. ≤ 3 levels)

### TanStack Query mistakes

```tsx
// === Query configuration mistakes ===

// ❌ queryKey omits inputs
function BadQuery({ userId, filters }) {
  const { data } = useQuery({
    queryKey: ['users'],  // missing userId and filters!
    queryFn: () => fetchUsers(userId, filters),
  });
  // data won’t refresh when userId or filters change
}

// ✅ queryKey includes every input that affects the result
function GoodQuery({ userId, filters }) {
  const { data } = useQuery({
    queryKey: ['users', userId, filters],
    queryFn: () => fetchUsers(userId, filters),
  });
}

// ❌ staleTime: 0 causes noisy refetching
const { data } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  // default staleTime: 0 — refetch on mount/focus
});

// ✅ Tune staleTime
const { data } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  staleTime: 5 * 60 * 1000,  // skip auto refetch for 5 minutes
});

// === useSuspenseQuery mistakes ===

// ❌ useSuspenseQuery + enabled (not supported)
const { data } = useSuspenseQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  enabled: !!userId,  // error — useSuspenseQuery has no enabled
});

// ✅ Conditional rendering instead
function UserQuery({ userId }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return <UserProfile user={data} />;
}

function Parent({ userId }) {
  if (!userId) return <SelectUser />;
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserQuery userId={userId} />
    </Suspense>
  );
}

// === Mutation mistakes ===

// ❌ No invalidation after success
const mutation = useMutation({
  mutationFn: updateUser,
  // forgot invalidate — UI shows stale cache
});

// ✅ Invalidate related queries on success
const mutation = useMutation({
  mutationFn: updateUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});

// ❌ Optimistic update without rollback
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    // no snapshot — can’t restore on failure!
  },
});

// ✅ Full optimistic flow
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });
    const previous = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    return { previous };
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});

// === v5 migration mistakes ===

// ❌ Deprecated tuple API
const { data, isLoading } = useQuery(['key'], fetchFn);  // v4 style

// ✅ v5 object argument
const { data, isPending } = useQuery({
  queryKey: ['key'],
  queryFn: fetchFn,
});

// ❌ Confusing isPending vs isLoading
if (isLoading) return <Spinner />;
// v5: isLoading === isPending && isFetching

// ✅ Pick the flag that matches intent
if (isPending) return <Spinner />;  // no cached data yet
// or
if (isFetching) return <Refreshing />;  // background refresh
```

### TanStack Query Checklist
- [ ] `queryKey` includes every parameter that affects the query
- [ ] Reasonable `staleTime` (not always default 0)
- [ ] No `enabled` with `useSuspenseQuery`
- [ ] Mutations invalidate or update affected queries
- [ ] Optimistic updates snapshot and roll back on error
- [ ] v5 object-argument `useQuery` / `useMutation` API
- [ ] Clear mental model: `isPending` vs `isLoading` vs `isFetching`

### TypeScript/JavaScript Common Mistakes
- [ ] `==` instead of `===`
- [ ] Modifying array/object during iteration
- [ ] `this` context lost in callbacks
- [ ] Missing `key` prop in lists
- [ ] Closure capturing loop variable
- [ ] parseInt without radix parameter

## Vue 3

### Lost reactivity
```vue
<!-- ❌ Destructuring reactive loses reactivity -->
<script setup>
const state = reactive({ count: 0 })
const { count } = state  // count is not reactive!
</script>

<!-- ✅ Use toRefs -->
<script setup>
const state = reactive({ count: 0 })
const { count } = toRefs(state)  // count.value is reactive
</script>
```

### Passing reactive props into composables
```vue
<!-- ❌ Passing a plain prop value — composable won’t track updates -->
<script setup>
const props = defineProps<{ id: string }>()
const { data } = useFetch(props.id)  // won’t refetch when id changes!
</script>

<!-- ✅ toRef or getter -->
<script setup>
const props = defineProps<{ id: string }>()
const { data } = useFetch(() => props.id)  // getter stays reactive
// or
const { data } = useFetch(toRef(props, 'id'))
</script>
```

### Watch cleanup
```vue
<!-- ❌ Async watch without cleanup — races -->
<script setup>
watch(id, async (newId) => {
  const data = await fetchData(newId)
  result.value = data  // stale responses can overwrite newer ones!
})
</script>

<!-- ✅ onCleanup to abort in-flight work -->
<script setup>
watch(id, async (newId, _, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  const data = await fetchData(newId, controller.signal)
  result.value = data
})
</script>
```

### Side effects in `computed`
```vue
<!-- ❌ Mutating other state inside computed -->
<script setup>
const total = computed(() => {
  sideEffect.value++  // side effect — runs on every read
  return items.value.reduce((a, b) => a + b, 0)
})
</script>

<!-- ✅ Keep computed pure -->
<script setup>
const total = computed(() => {
  return items.value.reduce((a, b) => a + b, 0)
})
// side effects in watch
watch(total, () => { sideEffect.value++ })
</script>
```

### Common template mistakes
```vue
<!-- ❌ v-if and v-for on the same element (v-if wins — confusing) -->
<template>
  <div v-for="item in items" v-if="item.visible" :key="item.id">
    {{ item.name }}
  </div>
</template>

<!-- ✅ computed filter or outer template -->
<template>
  <template v-for="item in items" :key="item.id">
    <div v-if="item.visible">{{ item.name }}</div>
  </template>
</template>
```

### Common Mistakes
- [ ] Destructuring `reactive` without `toRefs` loses reactivity
- [ ] Props passed into composables not wrapped for reactivity
- [ ] Async `watch` without cleanup / abort
- [ ] Side effects inside `computed`
- [ ] `v-for` with index as `key` when the list reorders
- [ ] `v-if` and `v-for` on the same element
- [ ] `defineProps` without TypeScript types
- [ ] `withDefaults` object/array defaults without factory functions
- [ ] Mutating props instead of emitting
- [ ] `watchEffect` with unclear deps — runs too often

## Python

### Mutable Default Arguments
```python
# ❌ Bug: List shared across all calls
def add_item(item, items=[]):
    items.append(item)
    return items

# ✅ Correct
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### Exception Handling
```python
# ❌ Catching everything, including KeyboardInterrupt
try:
    risky_operation()
except:
    pass

# ✅ Catch specific exceptions
try:
    risky_operation()
except ValueError as e:
    logger.error(f"Invalid value: {e}")
    raise
```

### Class Attributes
```python
# ❌ Shared mutable class attribute
class User:
    permissions = []  # Shared across all instances!

# ✅ Initialize in __init__
class User:
    def __init__(self):
        self.permissions = []
```

### Common Mistakes
- [ ] Using `is` instead of `==` for value comparison
- [ ] Forgetting `self` parameter in methods
- [ ] Modifying list while iterating
- [ ] String concatenation in loops (use join)
- [ ] Not closing files (use `with` statement)

## Rust

### Ownership and borrowing

```rust
// ❌ Use after move
let s = String::from("hello");
let s2 = s;
println!("{}", s);  // Error: s was moved

// ✅ Clone if needed (but consider if clone is necessary)
let s = String::from("hello");
let s2 = s.clone();
println!("{}", s);  // OK

// ❌ clone() to silence the borrow checker (anti-pattern)
fn process(data: &Data) {
    let owned = data.clone();  // unnecessary clone
    do_something(owned);
}

// ✅ Prefer borrowing through the API
fn process(data: &Data) {
    do_something(data);  // pass reference
}

// ❌ Storing references in structs (often painful)
struct Parser<'a> {
    input: &'a str,  // complicates lifetimes
    position: usize,
}

// ✅ Owned data
struct Parser {
    input: String,  // owned — simpler lifetimes
    position: usize,
}

// ❌ Mutating a collection while iterating
let mut vec = vec![1, 2, 3];
for item in &vec {
    vec.push(*item);  // Error: cannot borrow as mutable
}

// ✅ Build a new collection
let vec = vec![1, 2, 3];
let new_vec: Vec<_> = vec.iter().map(|x| x * 2).collect();
```

### Reviewing `unsafe` code

```rust
// ❌ unsafe block with no safety comment
unsafe {
    ptr::write(dest, value);
}

// ✅ SAFETY comment documenting invariants
// SAFETY: `dest` comes from `Vec::as_mut_ptr()` and we guarantee:
// 1. Valid, aligned pointer
// 2. Target memory is not aliased by other references
// 3. Write stays within allocated capacity
unsafe {
    ptr::write(dest, value);
}

// ❌ unsafe fn without # Safety docs
pub unsafe fn from_raw_parts(ptr: *mut T, len: usize) -> Self { ... }

// ✅ Document the safety contract
/// Creates a new instance from raw parts.
///
/// # Safety
///
/// - `ptr` must have been allocated via `GlobalAlloc`
/// - `len` must be less than or equal to the allocated capacity
/// - The caller must ensure no other references to the memory exist
pub unsafe fn from_raw_parts(ptr: *mut T, len: usize) -> Self { ... }

// ❌ Splitting unsafe invariants across modules
mod a {
    pub fn set_flag() { FLAG = true; }  // safe code affects unsafe assumptions
}
mod b {
    pub unsafe fn do_thing() {
        if FLAG { /* assumes FLAG means something */ }
    }
}

// ✅ Keep unsafe boundaries in one module
mod safe_wrapper {
    // all unsafe logic lives here
    // expose a safe API outward
}
```

### Async / concurrency

```rust
// ❌ Blocking inside async
async fn bad_fetch(url: &str) -> Result<String> {
    let resp = reqwest::blocking::get(url)?;  // blocks the runtime
    Ok(resp.text()?)
}

// ✅ Async I/O
async fn good_fetch(url: &str) -> Result<String> {
    let resp = reqwest::get(url).await?;
    Ok(resp.text().await?)
}

// ❌ Holding a Mutex across `.await`
async fn bad_lock(mutex: &Mutex<Data>) {
    let guard = mutex.lock().unwrap();
    some_async_op().await;  // lock held across await!
    drop(guard);
}

// ✅ Minimize critical sections
async fn good_lock(mutex: &Mutex<Data>) {
    let data = {
        let guard = mutex.lock().unwrap();
        guard.clone()  // release lock before await
    };
    some_async_op().await;
    // use data
}

// ❌ std::sync::Mutex held across await in async fn
async fn bad_async_mutex(mutex: &std::sync::Mutex<Data>) {
    let _guard = mutex.lock().unwrap();  // can deadlock the runtime
    tokio::time::sleep(Duration::from_secs(1)).await;
}

// ✅ tokio::sync::Mutex when you must await while locked
async fn good_async_mutex(mutex: &tokio::sync::Mutex<Data>) {
    let _guard = mutex.lock().await;
    tokio::time::sleep(Duration::from_secs(1)).await;
}

// ❌ Forgetting futures are lazy
fn bad_spawn() {
    let future = async_operation();  // not running yet
    // dropped — nothing happens
}

// ✅ `.await` or spawn
async fn good_spawn() {
    async_operation().await;
    // or
    tokio::spawn(async_operation());
}

// ❌ Spawned task not `'static`
async fn bad_spawn_lifetime(data: &str) {
    tokio::spawn(async {
        println!("{}", data);  // Error: `data` is not 'static
    });
}

// ✅ `move` + owned data (or `Arc`)
async fn good_spawn_lifetime(data: String) {
    tokio::spawn(async move {
        println!("{}", data);  // OK — owned
    });
}
```

### Error handling

```rust
// ❌ unwrap/expect in production paths
fn bad_parse(input: &str) -> i32 {
    input.parse().unwrap()  // panic!
}

// ✅ Propagate errors
fn good_parse(input: &str) -> Result<i32, ParseIntError> {
    input.parse()
}

// ❌ Dropping error context
fn bad_error_handling() -> Result<()> {
    match operation() {
        Ok(v) => Ok(v),
        Err(_) => Err(anyhow!("operation failed"))  // loses cause
    }
}

// ✅ Add context
fn good_error_handling() -> Result<()> {
    operation().context("failed to perform operation")?;
    Ok(())
}

// ❌ Libraries returning anyhow (prefer thiserror)
// lib.rs
pub fn parse_config(path: &str) -> anyhow::Result<Config> {
    // callers can’t match on error kinds
}

// ✅ Library error enums with thiserror
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("failed to read config file: {0}")]
    Io(#[from] std::io::Error),
    #[error("invalid config format: {0}")]
    Parse(#[from] serde_json::Error),
}

pub fn parse_config(path: &str) -> Result<Config, ConfigError> {
    // callers can match variants
}

// ❌ Ignoring `#[must_use]` results
fn bad_ignore_result() {
    some_fallible_operation();  // warning: unused Result
}

// ✅ Handle or explicitly discard
fn good_handle_result() {
    let _ = some_fallible_operation();  // explicit ignore
    // or
    some_fallible_operation().ok();  // convert to Option
}
```

### Performance pitfalls

```rust
// ❌ Unnecessary collect
fn bad_process(items: &[i32]) -> i32 {
    items.iter()
        .filter(|x| **x > 0)
        .collect::<Vec<_>>()  // extra allocation
        .iter()
        .sum()
}

// ✅ Lazy iterator chain
fn good_process(items: &[i32]) -> i32 {
    items.iter()
        .filter(|x| **x > 0)
        .sum()
}

// ❌ Repeated allocation in a loop
fn bad_loop() -> String {
    let mut result = String::new();
    for i in 0..1000 {
        result = result + &i.to_string();  // realloc every iteration
    }
    result
}

// ✅ Preallocate or append in place
fn good_loop() -> String {
    let mut result = String::with_capacity(4000);
    for i in 0..1000 {
        write!(result, "{}", i).unwrap();  // in-place append
    }
    result
}

// ❌ Excessive clone
fn bad_clone(data: &HashMap<String, Vec<u8>>) -> Vec<u8> {
    data.get("key").cloned().unwrap_or_default()
}

// ✅ Return a reference or use Cow
fn good_ref(data: &HashMap<String, Vec<u8>>) -> &[u8] {
    data.get("key").map(|v| v.as_slice()).unwrap_or(&[])
}

// ❌ Large struct by value
fn bad_pass(data: LargeStruct) { ... }  // copies whole struct

// ✅ Pass by reference
fn good_pass(data: &LargeStruct) { ... }

// ❌ Box<dyn Trait> for a small, known type
fn bad_trait_object() -> Box<dyn Iterator<Item = i32>> {
    Box::new(vec![1, 2, 3].into_iter())
}

// ✅ impl Trait
fn good_impl_trait() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}

// ❌ retain can be slower than filter+collect in some cases
vec.retain(|x| x.is_valid());  // O(n) with a larger constant factor

// ✅ filter + collect when in-place mutation isn’t required
let vec: Vec<_> = vec.into_iter().filter(|x| x.is_valid()).collect();
```

### Lifetimes and references

```rust
// ❌ Returning a reference to a local
fn bad_return_ref() -> &str {
    let s = String::from("hello");
    &s  // Error: s will be dropped
}

// ✅ Return owned data or a `'static` reference
fn good_return_owned() -> String {
    String::from("hello")
}

// ❌ Over-generic lifetimes
fn bad_lifetime<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x  // 'b is unused
}

// ✅ Let the compiler elide
fn good_lifetime(x: &str, _y: &str) -> &str {
    x
}

// ❌ Related fields with unrelated lifetimes
struct Bad<'a, 'b> {
    name: &'a str,
    data: &'b [u8],  // usually should share one lifetime
}

// ✅ One lifetime for related borrows
struct Good<'a> {
    name: &'a str,
    data: &'a [u8],
}
```

### Rust review checklist

**Ownership and borrowing**
- [ ] `clone()` is intentional, not a blanket escape hatch
- [ ] Avoid storing references in structs unless necessary
- [ ] `Rc`/`Arc` used deliberately — no hidden shared mutable state
- [ ] No unnecessary `RefCell` (runtime checks vs compile-time)

**Unsafe**
- [ ] Every `unsafe` block has a SAFETY comment
- [ ] `unsafe fn` documents `# Safety`
- [ ] Invariants are written down
- [ ] Unsafe surface area is as small as possible

**Async / concurrency**
- [ ] No blocking calls inside async contexts
- [ ] No `std::sync` locks held across `.await`
- [ ] Spawned tasks satisfy `'static`
- [ ] Futures are actually `.await`ed or spawned
- [ ] Consistent lock ordering (avoid deadlocks)

**Errors**
- [ ] Libraries: `thiserror`; applications: often `anyhow`
- [ ] Errors carry enough context
- [ ] No `unwrap`/`expect` on fallible production paths
- [ ] `#[must_use]` results handled or explicitly discarded

**Performance**
- [ ] Avoid gratuitous `collect()`
- [ ] Large values passed by reference
- [ ] String building uses `with_capacity` / `write!` when hot
- [ ] Prefer `impl Trait` over `Box<dyn Trait>` when possible

**Types**
- [ ] Newtypes for extra safety where it helps
- [ ] Exhaustive matches on enums — no catch-all `_` hiding new variants
- [ ] Lifetimes kept as simple as possible

## SQL

### Injection Vulnerabilities
```sql
-- ❌ String concatenation (SQL injection risk)
query = "SELECT * FROM users WHERE id = " + user_id

-- ✅ Parameterized queries
query = "SELECT * FROM users WHERE id = ?"
cursor.execute(query, (user_id,))
```

### Performance Issues
- [ ] Missing indexes on filtered/joined columns
- [ ] SELECT * instead of specific columns
- [ ] N+1 query patterns
- [ ] Missing LIMIT on large tables
- [ ] Inefficient subqueries vs JOINs

### Common Mistakes
- [ ] Not handling NULL comparisons correctly
- [ ] Missing transactions for related operations
- [ ] Incorrect JOIN types
- [ ] Case sensitivity issues
- [ ] Date/timezone handling errors

## API Design

### REST Issues
- [ ] Inconsistent resource naming
- [ ] Wrong HTTP methods (POST for idempotent operations)
- [ ] Missing pagination for list endpoints
- [ ] Incorrect status codes
- [ ] Missing rate limiting

### Data Validation
- [ ] Missing input validation
- [ ] Incorrect data type validation
- [ ] Missing length/range checks
- [ ] Not sanitizing user input
- [ ] Trusting client-side validation

## Testing

### Test Quality Issues
- [ ] Testing implementation details instead of behavior
- [ ] Missing edge case tests
- [ ] Flaky tests (non-deterministic)
- [ ] Tests with external dependencies
- [ ] Missing negative tests (error cases)
- [ ] Overly complex test setup
