# TypeScript/JavaScript Code Review Guide

> TypeScript review guide covering the type system, generics, conditional types, strict mode, and async/await patterns.

## Contents

- [Type-safety basics](#type-safety-basics)
- [Generic patterns](#generic-patterns)
- [Advanced types](#advanced-types)
- [Strict mode](#strict-mode)
- [Async patterns](#async-patterns)
- [Immutability](#immutability)
- [ESLint rules](#eslint-rules)
- [Review Checklist](#review-checklist)

---

## Type-safety basics

### Avoid `any`

```typescript
// ❌ Using any defeats type safety
function processData(data: any) {
  return data.value;  // no static checks — can fail at runtime
}

// ✅ Use proper types
interface DataPayload {
  value: string;
}
function processData(data: DataPayload) {
  return data.value;
}

// ✅ Use unknown + narrowing for unknown input
function processUnknown(data: unknown) {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return (data as { value: string }).value;
  }
  throw new Error('Invalid data');
}
```

### Narrowing types

```typescript
// ❌ Unsafe assertion
function getLength(value: string | string[]) {
  return (value as string[]).length;  // wrong if value is a string
}

// ✅ Type guards
function getLength(value: string | string[]): number {
  if (Array.isArray(value)) {
    return value.length;
  }
  return value.length;
}

// ✅ `in` operator
interface Dog { bark(): void }
interface Cat { meow(): void }

function speak(animal: Dog | Cat) {
  if ('bark' in animal) {
    animal.bark();
  } else {
    animal.meow();
  }
}
```

### Literal types and `as const`

```typescript
// ❌ Types too wide
const config = {
  endpoint: '/api',
  method: 'GET'  // typed as string
};

// ✅ as const for literal types
const config = {
  endpoint: '/api',
  method: 'GET'
} as const;  // method is 'GET'

// ✅ For function arguments
function request(method: 'GET' | 'POST', url: string) { ... }
request(config.method, config.endpoint);  // OK
```

---

## Generic patterns

### Basic generics

```typescript
// ❌ Duplication
function getFirstString(arr: string[]): string | undefined {
  return arr[0];
}
function getFirstNumber(arr: number[]): number | undefined {
  return arr[0];
}

// ✅ Single generic
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

### Generic constraints

```typescript
// ❌ Unconstrained T — can’t index safely
function getProperty<T>(obj: T, key: string) {
  return obj[key];  // Error: no index signature
}

// ✅ keyof constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'Alice', age: 30 };
getProperty(user, 'name');  // string
getProperty(user, 'age');   // number
getProperty(user, 'foo');   // Error: 'foo' is not keyof User
```

### Generic defaults

```typescript
// ✅ Sensible default type params
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

// Omit type argument
const response: ApiResponse = { data: null, status: 200, message: 'OK' };
// Or specify
const userResponse: ApiResponse<User> = { ... };
```

### Built-in utility types

```typescript
// ✅ Prefer built-in utilities
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;         // all optional
type RequiredUser = Required<User>;       // all required
type ReadonlyUser = Readonly<User>;       // all readonly
type UserKeys = keyof User;               // 'id' | 'name' | 'email'
type NameOnly = Pick<User, 'name'>;       // { name: string }
type WithoutId = Omit<User, 'id'>;        // { name: string; email: string }
type UserRecord = Record<string, User>;   // { [key: string]: User }
```

---

## Advanced types

### Conditional types

```typescript
// ✅ Different output types from input types
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// ✅ Element type of an array
type ElementType<T> = T extends (infer U)[] ? U : never;

type Elem = ElementType<string[]>;  // string

// ✅ Return type of a function (like built-in ReturnType)
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

### Mapped types

```typescript
// ✅ Map every property of an object type
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

interface User {
  name: string;
  age: number;
}

type NullableUser = Nullable<User>;
// { name: string | null; age: number | null }

// ✅ Key remapping with prefix
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

### Template literal types

```typescript
// ✅ Type-safe event names
type EventName = 'click' | 'focus' | 'blur';
type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur'

// ✅ API path pattern
type ApiRoute = `/api/${string}`;
const route: ApiRoute = '/api/users';  // OK
const badRoute: ApiRoute = '/users';   // Error
```

### Discriminated unions

```typescript
// ✅ Discriminant for safe narrowing
type Result<T, E> =
  | { success: true; data: T }
  | { success: false; error: E };

function handleResult(result: Result<User, Error>) {
  if (result.success) {
    console.log(result.data.name);  // TS knows data exists
  } else {
    console.log(result.error.message);  // TS knows error exists
  }
}

// ✅ Redux-style actions
type Action =
  | { type: 'INCREMENT'; payload: number }
  | { type: 'DECREMENT'; payload: number }
  | { type: 'RESET' };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case 'INCREMENT':
      return state + action.payload;  // payload narrowed
    case 'DECREMENT':
      return state - action.payload;
    case 'RESET':
      return 0;  // no payload here
  }
}
```

---

## Strict mode

### Recommended `tsconfig.json`

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,

    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noPropertyAccessFromIndexSignature": true
  }
}
```

### Effect of `noUncheckedIndexedAccess`

```typescript
// tsconfig: "noUncheckedIndexedAccess": true

const arr = [1, 2, 3];
const first = arr[0];  // number | undefined

// ❌ Unsafe direct use
console.log(first.toFixed(2));  // Error: may be undefined

// ✅ Narrow first
if (first !== undefined) {
  console.log(first.toFixed(2));
}

// ✅ Or non-null assertion when you know it’s there
console.log(arr[0]!.toFixed(2));
```

---

## Async patterns

### Promise error handling

```typescript
// ❌ Not handling async errors
async function fetchUser(id: string) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();  // network / HTTP errors ignored
}

// ✅ Handle errors properly
async function fetchUser(id: string): Promise<User> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return await response.json();
  } catch (error) {
    if (error instanceof Error) {
      throw new Error(`Failed to fetch user: ${error.message}`);
    }
    throw error;
  }
}
```

### `Promise.all` vs `Promise.allSettled`

```typescript
// ❌ Promise.all — one rejection fails everything
async function fetchAllUsers(ids: string[]) {
  const users = await Promise.all(ids.map(fetchUser));
  return users;
}

// ✅ allSettled — inspect each outcome
async function fetchAllUsers(ids: string[]) {
  const results = await Promise.allSettled(ids.map(fetchUser));

  const users: User[] = [];
  const errors: Error[] = [];

  for (const result of results) {
    if (result.status === 'fulfilled') {
      users.push(result.value);
    } else {
      errors.push(result.reason);
    }
  }

  return { users, errors };
}
```

### Race conditions

```typescript
// ❌ Stale responses can overwrite newer ones
function useSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(setResults);  // slower older request may win
  }, [query]);
}

// ✅ AbortController
function useSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(e => {
        if (e.name !== 'AbortError') throw e;
      });

    return () => controller.abort();
  }, [query]);
}
```

---

## Immutability

### `Readonly` and `ReadonlyArray`

```typescript
// ❌ Mutable array param accidentally mutated
function processUsers(users: User[]) {
  users.sort((a, b) => a.name.localeCompare(b.name));  // mutates caller’s array!
  return users;
}

// ✅ readonly + copy before sort
function processUsers(users: readonly User[]): User[] {
  return [...users].sort((a, b) => a.name.localeCompare(b.name));
}

// ✅ Deep readonly helper
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

### Immutable-style arguments

```typescript
// ✅ as const + readonly tuple
function createConfig<T extends readonly string[]>(routes: T) {
  return routes;
}

const routes = createConfig(['home', 'about', 'contact'] as const);
// readonly ['home', 'about', 'contact']
```

---

## ESLint rules

### Recommended `@typescript-eslint` setup

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:@typescript-eslint/recommended-requiring-type-checking',
    'plugin:@typescript-eslint/strict'
  ],
  rules: {
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unsafe-assignment': 'error',
    '@typescript-eslint/no-unsafe-member-access': 'error',
    '@typescript-eslint/no-unsafe-call': 'error',
    '@typescript-eslint/no-unsafe-return': 'error',

    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-floating-promises': 'error',
    '@typescript-eslint/await-thenable': 'error',
    '@typescript-eslint/no-misused-promises': 'error',

    '@typescript-eslint/consistent-type-imports': 'error',
    '@typescript-eslint/prefer-nullish-coalescing': 'error',
    '@typescript-eslint/prefer-optional-chain': 'error'
  }
};
```

### Fixing common ESLint issues

```typescript
// ❌ no-floating-promises — Promise must be handled
async function save() { ... }
save();  // Error: floating promise

// ✅ Handle explicitly
await save();
// or
save().catch(console.error);
// or intentional fire-and-forget
void save();

// ❌ no-misused-promises — async forEach is wrong
const items = [1, 2, 3];
items.forEach(async (item) => {  // Error!
  await processItem(item);
});

// ✅ for...of
for (const item of items) {
  await processItem(item);
}
// or Promise.all
await Promise.all(items.map(processItem));
```

---

## Review Checklist

### Type system
- [ ] No `any` (use `unknown` + guards instead)
- [ ] Interfaces/types are complete and well named
- [ ] Generics used for real reuse, not noise
- [ ] Unions narrowed correctly
- [ ] Utility types used where they help (`Partial`, `Pick`, `Omit`, …)

### Generics
- [ ] Constraints (`extends`) where needed
- [ ] Sensible defaults for type parameters
- [ ] Avoid over-generic APIs (KISS)

### Strict mode
- [ ] `strict: true` in `tsconfig.json`
- [ ] `noUncheckedIndexedAccess` enabled
- [ ] Prefer `@ts-expect-error` over `@ts-ignore` with a comment

### Async code
- [ ] `async` functions handle errors
- [ ] Promise rejections not ignored
- [ ] No floating promises
- [ ] `Promise.all` vs `allSettled` chosen on purpose
- [ ] Races mitigated (e.g. `AbortController`)

### Immutability
- [ ] Don’t mutate caller-owned collections/objects
- [ ] Prefer spread for new object/array snapshots
- [ ] Consider `readonly` on inputs

### ESLint
- [ ] `@typescript-eslint` recommended (+ type-aware as needed)
- [ ] No ignored warnings/errors without justification
- [ ] `consistent-type-imports` for type-only imports
