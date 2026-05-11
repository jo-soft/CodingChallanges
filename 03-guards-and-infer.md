# Challenge 3: More Type Guards & Infer

## Task Description

### Level 1: Narrowing the Union

In Angular we often model component state with a union (e.g. when fetching data from a service). We need to handle this state in a type-safe way.

Implement `handleState` so that it returns:
- `data.length` for a `Success` state
- `progress` for a `Loading` state
- `message` for an `Error` state

Provide two different approaches:
- **Solution A** – use standard control-flow (`switch/case` or `if/else`)
- **Solution B** – implement a custom type guard (e.g. `isSuccess`) and use it to narrow the type

```ts
interface StateLoading { type: 'loading'; progress: number }
interface StateSuccess { type: 'success'; data: string[] }
interface StateError   { type: 'error';   message: string }

type RemoteState = StateLoading | StateSuccess | StateError;

function handleState(state: RemoteState) {
  // Implement Solution A & B here
}
```

### Level 2: Extracting the Branch

Sometimes you don't need the whole union — only one specific branch of it (e.g. for a child component or a helper function).

Show two ways to define `OnlySuccess`, which extracts `StateSuccess` from `RemoteState`:
- **Solution A** – use the built-in TypeScript utility `Extract<T, U>`
- **Solution B** – create your own generic utility `MyExtract<T, V>` using a conditional type

### Level 3: The `infer` Constraint

We want to extract the type of the `data` property from a function's return type. The constraint is that the return type must contain a branch with `{ type: 'success'; data: T }`.

Implement `InferData<T>` such that it:
1. Extracts the `{ type: 'success'; ... }` branch from the return type (if it exists).
2. Then extracts the `data` property from that branch.

```ts
// Test cases:
type FetchUser    = () => { type: 'success'; data: { name: string } };
type FetchProduct = () => { data: { id: number; price: number } };

// Both should extract the data type from the success branch:
type User    = InferData<FetchUser>;    // { name: string }

// Should result in 'never' — no success branch with data in return type:
type Product = InferData<FetchProduct>; // type missing
type Invalid = InferData<() => string>; // return type does not match at all
```

**Note:** The key is to first filter for `{ type: 'success' }` before trying to extract `data`.

---

## Solutions

### Level 1: Narrowing the Union

**Solution A – switch/case**

TypeScript performs exhaustive narrowing on discriminated unions via `switch`. After each `case` branch, the type is fully narrowed to the matching member.

```ts
function handleStateA(state: RemoteState): number | string {
  switch (state.type) {
    case 'loading': return state.progress;
    case 'success': return state.data.length;
    case 'error':   return state.message;
  }
}
```

**Solution B – custom type guards**

Here the parameter is widened to `unknown` intentionally. The type guards do the full runtime validation, so the function is safe even when called from untyped code.

```ts
function isLoadingState(state: unknown): state is StateLoading {
  return typeof state === 'object' && state !== null &&
    'type' in state && state.type === 'loading';
}

function isSuccessState(state: unknown): state is StateSuccess {
  return typeof state === 'object' && state !== null &&
    'type' in state && state.type === 'success';
}

function isErrorState(state: unknown): state is StateError {
  return typeof state === 'object' && state !== null &&
    'type' in state && state.type === 'error';
}

function handleStateB(state: unknown): number | string | undefined {
  if (isLoadingState(state))  return state.progress;
  if (isSuccessState(state))  return state.data.length;
  if (isErrorState(state))    return state.message;
}
```

**Note:** Solution A is idiomatic when you control the input type. Solution B is preferable at system boundaries (e.g. parsing API responses) where the input truly is `unknown`.

---

### Level 2: Extracting the Branch

**Solution A – built-in `Extract`**

```ts
type OnlySuccess = Extract<RemoteState, { type: 'success' }>;
// Equivalent to: StateSuccess
```

**Solution B – custom `MyExtract` using a conditional type**

```ts
type MyExtract<T, V> = T extends V ? T : never;

type OnlySuccessCustom = MyExtract<RemoteState, { type: 'success' }>;
```

`MyExtract` distributes over the union `T`: for each member it asks "does this member extend `V`?". Those that do are kept; the rest become `never` and are eliminated from the result union.

---

### Level 3: The `infer` Constraint

The goal is to extract the `data` property from the `{ type: 'success' }` branch of a function's return type.

**Solution: Extract the success branch, then infer data**

```ts
type InferData<T extends (...args: any) => any> =
    Extract<ReturnType<T>, { type: 'success' }> extends { data: infer U }
        ? U extends unknown ? never: U
        : never;

type FetchUser = () => { type: 'success'; data: { name: string } };
type FetchProduct = () => { data: { id: number; price: number } };

type User = InferData<FetchUser>; // { name: string }
type Product = InferData<FetchProduct>; // never (no { type: 'success' } branch)
type Invalid = InferData<() => string>; // never (no { type: 'success' } branch)
```

**How it works:**

1. `Extract<ReturnType<T>, { type: 'success' }>` filters the return type to only the branch(es) that have `type: 'success'`.
2. If the extracted branch extends `{ data: infer D }`, we infer `D` (the data type).
3. If `D` is `unknown`, we return `never` to indicate that the data type is not properly defined. Otherwise, we return `D`. 
4. Otherwise, we return `never` (no matching success branch found).

**Key Insight:** This approach ensures that `InferData` only works with functions that have a `{ type: 'success'; data: X }` branch in their return type. Any function without this structure results in `never`.
