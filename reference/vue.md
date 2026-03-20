# Vue 3 Code Review Guide

> Code review guide for Vue 3 Composition API, covering the reactivity system, props/emits, watchers, composables, Vue 3.5 features, and related topics.

## Contents

- [Reactivity system](#reactivity-system)
- [Props & Emits](#props-emits)
- [Vue 3.5 features](#vue-35-features)
- [Watchers](#watchers)
- [Template best practices](#template-best-practices)
- [Composables](#composables)
- [Performance](#performance)
- [Review Checklist](#review-checklist)

---

## Reactivity system

### Choosing `ref` vs `reactive`

```vue
<!-- ✅ Use ref for primitives -->
<script setup lang="ts">
const count = ref(0)
const name = ref('Vue')

// ref needs .value
count.value++
</script>

<!-- ✅ Use reactive for objects/arrays (optional) -->
<script setup lang="ts">
const state = reactive({
  user: null,
  loading: false,
  error: null
})

// reactive: direct property access
state.loading = true
</script>

<!-- 💡 Modern practice: use ref everywhere for consistency -->
<script setup lang="ts">
const user = ref<User | null>(null)
const loading = ref(false)
const error = ref<Error | null>(null)
</script>
```

### Destructuring `reactive` objects

```vue
<!-- ❌ Destructuring reactive loses reactivity -->
<script setup lang="ts">
const state = reactive({ count: 0, name: 'Vue' })
const { count, name } = state  // reactivity lost!
</script>

<!-- ✅ Use toRefs to keep reactivity -->
<script setup lang="ts">
const state = reactive({ count: 0, name: 'Vue' })
const { count, name } = toRefs(state)  // stays reactive
// or use ref directly
const count = ref(0)
const name = ref('Vue')
</script>
```

### Side effects in `computed`

```vue
<!-- ❌ Side effects inside computed -->
<script setup lang="ts">
const fullName = computed(() => {
  console.log('Computing...')  // side effect!
  otherRef.value = 'changed'   // mutating other state!
  return `${firstName.value} ${lastName.value}`
})
</script>

<!-- ✅ computed is for derived state only -->
<script setup lang="ts">
const fullName = computed(() => {
  return `${firstName.value} ${lastName.value}`
})
// put side effects in watch or event handlers
watch(fullName, (name) => {
  console.log('Name changed:', name)
})
</script>
```

### `shallowRef` optimization

```vue
<!-- ❌ ref on large objects is deeply reactive -->
<script setup lang="ts">
const largeData = ref(hugeNestedObject)  // deep reactivity, higher cost
</script>

<!-- ✅ shallowRef skips deep conversion -->
<script setup lang="ts">
const largeData = shallowRef(hugeNestedObject)

// only full replacement triggers updates
function updateData(newData) {
  largeData.value = newData  // ✅ triggers update
}

// ❌ mutating nested fields does not trigger updates
// largeData.value.nested.prop = 'new'

// use triggerRef when you must mutate in place
import { triggerRef } from 'vue'
largeData.value.nested.prop = 'new'
triggerRef(largeData)
</script>
```

---

## Props & Emits

### Mutating props directly

```vue
<!-- ❌ Mutating props -->
<script setup lang="ts">
const props = defineProps<{ user: User }>()
props.user.name = 'New Name'  // never mutate props directly!
</script>

<!-- ✅ Emit so the parent can update -->
<script setup lang="ts">
const props = defineProps<{ user: User }>()
const emit = defineEmits<{
  update: [name: string]
}>()
const updateName = (name: string) => emit('update', name)
</script>
```

### Typing `defineProps`

```vue
<!-- ❌ defineProps without types -->
<script setup lang="ts">
const props = defineProps(['title', 'count'])  // no type checking
</script>

<!-- ✅ Typed props + withDefaults -->
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  items?: string[]
}
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => []  // objects/arrays need factory defaults
})
</script>
```

### Type-safe `defineEmits`

```vue
<!-- ❌ defineEmits without types -->
<script setup lang="ts">
const emit = defineEmits(['update', 'delete'])  // no type checking
emit('update', someValue)  // unsafe payload types
</script>

<!-- ✅ Full typing -->
<script setup lang="ts">
const emit = defineEmits<{
  update: [id: number, value: string]
  delete: [id: number]
  'custom-event': [payload: CustomPayload]
}>()

// full type checking
emit('update', 1, 'new value')  // ✅
emit('update', 'wrong')  // ❌ TypeScript error
</script>
```

---

## Vue 3.5 features

### Reactive props destructure (3.5+)

```vue
<!-- Before Vue 3.5: destructuring lost reactivity -->
<script setup lang="ts">
const props = defineProps<{ count: number }>()
// use props.count or toRefs
</script>

<!-- ✅ Vue 3.5+: destructure stays reactive -->
<script setup lang="ts">
const { count, name = 'default' } = defineProps<{
  count: number
  name?: string
}>()

// count and name stay reactive
// safe to use in template and watch
watch(() => count, (newCount) => {
  console.log('Count changed:', newCount)
})
</script>

<!-- ✅ With defaults -->
<script setup lang="ts">
const {
  title,
  count = 0,
  items = () => []  // function default for object/array
} = defineProps<{
  title: string
  count?: number
  items?: () => string[]
}>()
</script>
```

### `defineModel` (3.4+)

```vue
<!-- ❌ Classic v-model: verbose -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

// often need computed for two-way binding
const value = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})
</script>

<!-- ✅ defineModel: concise v-model -->
<script setup lang="ts">
// wires props and emit for you
const model = defineModel<string>()

// direct use
model.value = 'new value'  // emits automatically
</script>
<template>
  <input v-model="model" />
</template>

<!-- ✅ Named v-model -->
<script setup lang="ts">
// v-model:title
const title = defineModel<string>('title')

// default and options
const count = defineModel<number>('count', {
  default: 0,
  required: false
})
</script>

<!-- ✅ Multiple v-models -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
</script>
<template>
  <!-- Parent: <MyInput v-model:first-name="first" v-model:last-name="last" /> -->
</template>

<!-- ✅ v-model modifiers -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string>()

// inspect modifiers
if (modifiers.capitalize) {
  // handle .capitalize
}
</script>
```

### `useTemplateRef` (3.5+)

```vue
<!-- Classic: ref attribute matches variable name -->
<script setup lang="ts">
const inputRef = ref<HTMLInputElement | null>(null)
</script>
<template>
  <input ref="inputRef" />
</template>

<!-- ✅ useTemplateRef: clearer template refs -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'

const input = useTemplateRef<HTMLInputElement>('my-input')

onMounted(() => {
  input.value?.focus()
})
</script>
<template>
  <input ref="my-input" />
</template>

<!-- ✅ Dynamic ref -->
<script setup lang="ts">
const refKey = ref('input-a')
const dynamicInput = useTemplateRef<HTMLInputElement>(refKey)
</script>
```

### `useId` (3.5+)

```vue
<!-- ❌ Hand-rolled IDs can clash -->
<script setup lang="ts">
const id = `input-${Math.random()}`  // SSR mismatch!
</script>

<!-- ✅ useId: SSR-safe unique IDs -->
<script setup lang="ts">
import { useId } from 'vue'

const id = useId()  // e.g. 'v-0'
</script>
<template>
  <label :for="id">Name</label>
  <input :id="id" />
</template>

<!-- ✅ In form components -->
<script setup lang="ts">
const inputId = useId()
const errorId = useId()
</script>
<template>
  <label :for="inputId">Email</label>
  <input
    :id="inputId"
    :aria-describedby="errorId"
  />
  <span :id="errorId" class="error">{{ error }}</span>
</template>
```

### `onWatcherCleanup` (3.5+)

```vue
<!-- Classic: third argument to watch -->
<script setup lang="ts">
watch(source, async (value, oldValue, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())
  // ...
})
</script>

<!-- ✅ onWatcherCleanup: more flexible cleanup -->
<script setup lang="ts">
import { onWatcherCleanup } from 'vue'

watch(source, async (value) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())

  // can register cleanup anywhere in the callback
  if (someCondition) {
    const anotherResource = createResource()
    onWatcherCleanup(() => anotherResource.dispose())
  }

  await fetchData(value, controller.signal)
})
</script>
```

### Deferred `Teleport` (3.5+)

```vue
<!-- ❌ Teleport target must exist at mount -->
<template>
  <Teleport to="#modal-container">
    <!-- errors if #modal-container is missing -->
  </Teleport>
</template>

<!-- ✅ defer delays mounting until target exists -->
<template>
  <Teleport to="#modal-container" defer>
    <!-- mounts after the target appears -->
    <Modal />
  </Teleport>
</template>
```

---

## Watchers

### `watch` vs `watchEffect`

```vue
<script setup lang="ts">
// ✅ watch: explicit deps, lazy
watch(
  () => props.userId,
  async (userId) => {
    user.value = await fetchUser(userId)
  }
)

// ✅ watchEffect: auto-tracked, runs immediately
watchEffect(async () => {
  // tracks props.userId
  user.value = await fetchUser(props.userId)
})

// 💡 When to use which:
// - need old value? → watch
// - need lazy run? → watch
// - messy deps? → watchEffect
</script>
```

### Cleanup in `watch`

```vue
<!-- ❌ No cleanup: stale requests, leaks -->
<script setup lang="ts">
watch(searchQuery, async (query) => {
  const controller = new AbortController()
  const data = await fetch(`/api/search?q=${query}`, {
    signal: controller.signal
  })
  results.value = await data.json()
  // rapid query changes won't cancel older requests!
})
</script>

<!-- ✅ onCleanup for side effects -->
<script setup lang="ts">
watch(searchQuery, async (query, _, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())  // cancel previous request

  try {
    const data = await fetch(`/api/search?q=${query}`, {
      signal: controller.signal
    })
    results.value = await data.json()
  } catch (e) {
    if (e.name !== 'AbortError') throw e
  }
})
</script>
```

### `watch` options

```vue
<script setup lang="ts">
// ✅ immediate: run once up front
watch(
  userId,
  async (id) => {
    user.value = await fetchUser(id)
  },
  { immediate: true }
)

// ✅ deep: deep watch (expensive—use sparingly)
watch(
  state,
  (newState) => {
    console.log('State changed deeply')
  },
  { deep: true }
)

// ✅ flush: 'post': after DOM updates
watch(
  source,
  () => {
    // safe to read updated DOM
    // nextTick often unnecessary
  },
  { flush: 'post' }
)

// ✅ once: true (Vue 3.4+): fire once
watch(
  source,
  (value) => {
    console.log('runs only once:', value)
  },
  { once: true }
)
</script>
```

### Watching multiple sources

```vue
<script setup lang="ts">
// ✅ Multiple refs
watch(
  [firstName, lastName],
  ([newFirst, newLast], [oldFirst, oldLast]) => {
    console.log(`Name changed from ${oldFirst} ${oldLast} to ${newFirst} ${newLast}`)
  }
)

// ✅ Specific fields on a reactive object
watch(
  () => [state.count, state.name],
  ([count, name]) => {
    console.log(`count: ${count}, name: ${name}`)
  }
)
</script>
```

---

## Template best practices

### `key` with `v-for`

```vue
<!-- ❌ index as key -->
<template>
  <li v-for="(item, index) in items" :key="index">
    {{ item.name }}
  </li>
</template>

<!-- ✅ Stable unique key -->
<template>
  <li v-for="item in items" :key="item.id">
    {{ item.name }}
  </li>
</template>

<!-- ✅ Composite key when no single id -->
<template>
  <li v-for="(item, index) in items" :key="`${item.name}-${item.type}-${index}`">
    {{ item.name }}
  </li>
</template>
```

### `v-if` and `v-for` precedence

```vue
<!-- ❌ v-if and v-for on the same element -->
<template>
  <li v-for="user in users" v-if="user.active" :key="user.id">
    {{ user.name }}
  </li>
</template>

<!-- ✅ Filter with computed -->
<script setup lang="ts">
const activeUsers = computed(() =>
  users.value.filter(user => user.active)
)
</script>
<template>
  <li v-for="user in activeUsers" :key="user.id">
    {{ user.name }}
  </li>
</template>

<!-- ✅ Or wrap with template -->
<template>
  <template v-for="user in users" :key="user.id">
    <li v-if="user.active">
      {{ user.name }}
    </li>
  </template>
</template>
```

### Event handling

```vue
<!-- ❌ Heavy inline logic -->
<template>
  <button @click="items = items.filter(i => i.id !== item.id); count--">
    Delete
  </button>
</template>

<!-- ✅ Use methods -->
<script setup lang="ts">
const deleteItem = (id: number) => {
  items.value = items.value.filter(i => i.id !== id)
  count.value--
}
</script>
<template>
  <button @click="deleteItem(item.id)">Delete</button>
</template>

<!-- ✅ Event modifiers -->
<template>
  <!-- prevent default -->
  <form @submit.prevent="handleSubmit">...</form>

  <!-- stop propagation -->
  <button @click.stop="handleClick">...</button>

  <!-- run once -->
  <button @click.once="handleOnce">...</button>

  <!-- key modifiers -->
  <input @keyup.enter="submit" @keyup.esc="cancel" />
</template>
```

---

## Composables

### Composable design

```typescript
// ✅ Solid composable
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)

  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => count.value = initialValue

  return {
    count: readonly(count),  // readonly avoids external mutation
    increment,
    decrement,
    reset
  }
}

// ❌ Do not return .value
export function useBadCounter() {
  const count = ref(0)
  return {
    count: count.value  // ❌ loses reactivity
  }
}
```

### Passing props into composables

```vue
<!-- ❌ Plain prop value loses reactivity -->
<script setup lang="ts">
const props = defineProps<{ userId: string }>()
const { user } = useUser(props.userId)  // not reactive!
</script>

<!-- ✅ toRef or computed -->
<script setup lang="ts">
const props = defineProps<{ userId: string }>()
const userIdRef = toRef(props, 'userId')
const { user } = useUser(userIdRef)  // reactive
// or
const { user } = useUser(computed(() => props.userId))

// ✅ Vue 3.5+: destructure props
const { userId } = defineProps<{ userId: string }>()
const { user } = useUser(() => userId)  // getter
</script>
```

### Async composables

```typescript
// ✅ Async composable pattern
export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  const execute = async () => {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(toValue(url))
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`)
      }
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  // refetch when URL changes
  watchEffect(() => {
    toValue(url)  // track dependency
    execute()
  })

  return {
    data: readonly(data),
    error: readonly(error),
    loading: readonly(loading),
    refetch: execute
  }
}

// usage
const { data, loading, error, refetch } = useFetch<User[]>('/api/users')
```

### Lifecycle and cleanup

```typescript
// ✅ Lifecycle in composables
export function useEventListener(
  target: MaybeRefOrGetter<EventTarget>,
  event: string,
  handler: EventListener
) {
  onMounted(() => {
    toValue(target).addEventListener(event, handler)
  })

  onUnmounted(() => {
    toValue(target).removeEventListener(event, handler)
  })
}

// ✅ effectScope for grouped effects
export function useFeature() {
  const scope = effectScope()

  scope.run(() => {
    const state = ref(0)
    watch(state, () => { /* ... */ })
    watchEffect(() => { /* ... */ })
  })

  onUnmounted(() => scope.stop())

  return { /* ... */ }
}
```

---

## Performance

### `v-memo`

```vue
<!-- ✅ v-memo: skip re-rendering when deps unchanged -->
<template>
  <div v-for="item in list" :key="item.id" v-memo="[item.id === selected]">
    <!-- re-renders only when item.id === selected changes -->
    <ExpensiveComponent :item="item" :selected="item.id === selected" />
  </div>
</template>

<!-- ✅ With v-for -->
<template>
  <div
    v-for="item in list"
    :key="item.id"
    v-memo="[item.name, item.status]"
  >
    <!-- re-renders when name or status changes -->
  </div>
</template>
```

### `defineAsyncComponent`

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

// ✅ Lazy-load heavy UI
const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)

// ✅ Loading and error UI
const AsyncModal = defineAsyncComponent({
  loader: () => import('./components/Modal.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,  // delay showing loading (reduce flicker)
  timeout: 3000
})
</script>
```

### `KeepAlive`

```vue
<template>
  <!-- ✅ Cache dynamic components -->
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>

  <!-- ✅ Whitelist by name -->
  <KeepAlive include="TabA,TabB">
    <component :is="currentTab" />
  </KeepAlive>

  <!-- ✅ Cap cache size -->
  <KeepAlive :max="10">
    <component :is="currentTab" />
  </KeepAlive>
</template>

<script setup lang="ts">
// KeepAlive child hooks
onActivated(() => {
  // restored from cache
  refreshData()
})

onDeactivated(() => {
  // pushed into cache
  pauseTimers()
})
</script>
```

### Virtual lists

```vue
<!-- ✅ Virtual scroll for large lists -->
<script setup lang="ts">
import { useVirtualList } from '@vueuse/core'

const { list, containerProps, wrapperProps } = useVirtualList(
  items,
  { itemHeight: 50 }
)
</script>
<template>
  <div v-bind="containerProps" style="height: 400px; overflow: auto">
    <div v-bind="wrapperProps">
      <div v-for="item in list" :key="item.data.id" style="height: 50px">
        {{ item.data.name }}
      </div>
    </div>
  </div>
</template>
```

---

## Review Checklist

### Reactivity
- [ ] `ref` for primitives; `reactive` for objects—or standardize on `ref`
- [ ] No plain destructuring of `reactive` (or use `toRefs`)
- [ ] Props passed into composables stay reactive (`toRef`, getter, etc.)
- [ ] `shallowRef` / `shallowReactive` for large structures when appropriate
- [ ] No side effects inside `computed`

### Props & Emits
- [ ] `defineProps` typed with TypeScript
- [ ] Complex defaults use `withDefaults` + factories
- [ ] `defineEmits` fully typed
- [ ] Props never mutated in place
- [ ] Consider `defineModel` for v-model (Vue 3.4+)

### Vue 3.5 (when applicable)
- [ ] Reactive props destructure for simpler prop access
- [ ] `useTemplateRef` instead of string-matched `ref`
- [ ] `useId` for SSR-safe form IDs
- [ ] `onWatcherCleanup` for non-trivial watcher teardown

### Watchers
- [ ] `watch` / `watchEffect` register cleanup where needed
- [ ] Async watches handle races (abort, ignore stale results)
- [ ] `flush: 'post'` when touching the DOM after updates
- [ ] Avoid watcher overuse—prefer `computed` when possible
- [ ] Consider `once: true` for one-shot listeners

### Templates
- [ ] `v-for` uses stable, unique keys
- [ ] `v-if` and `v-for` not on the same element
- [ ] Event handlers use methods, not bulky inline code
- [ ] Virtual scrolling for very large lists

### Composables
- [ ] Related logic lives in composables
- [ ] Return refs, not raw `.value`
- [ ] Don't wrap trivial pure functions as composables
- [ ] Side effects cleaned up on unmount
- [ ] `effectScope` for grouped reactive effects when it helps

### Performance
- [ ] Split large components
- [ ] `defineAsyncComponent` for code splitting
- [ ] Avoid unnecessary deep reactivity
- [ ] `v-memo` for expensive list rows
- [ ] `KeepAlive` where tab-like views benefit from caching
