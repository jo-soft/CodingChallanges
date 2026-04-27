# Challenge 2: Type Guards

## Task Description

### Level 1: The Unknown Barrier
We receive data from an external source. The goal is to replace the unsafe type `any` with `unknown` and make the compiler "happy" again.
- Write a function that:
  - For a string: outputs "Text length: [number]"
  - For an array: outputs "List size: [number]"

```ts
function processInput(data: unknown): void {
  // ...your solution...
}
```

### Level 2: The Specialized Guard
Outsource the validation into a reusable function to increase type safety.
- Implement a type guard for `ApiResponse`.
- Use this guard to process the data accordingly.
- What is a type guard? What does `isApiResponse` do?

```ts
type ApiResponse = {
  data: string | string[];
  status: number;
};

function isApiResponse(obj: unknown) {
  // ...your solution...
}

function handleResponse(response: unknown) {
  if (isApiResponse(response)) {
    console.log(response.data);
  }
}
```

What is a type guard? What does `isApiResponse` do?

### Level 3: The Property Dilemma
Distinguish precisely between types with the same property names.
- Write a function `analyzeSubmission(input: Submission)` that:
  - Outputs the text in uppercase if it is a `TextDraft`.
  - Outputs the first element of the list if it is an `AttachmentList`.
  - How can you uniquely distinguish the types?

```ts
interface TextDraft {
  content: string;
  length: number;
}

interface AttachmentList {
  files: string[];
  length: number;
}

type Submission = TextDraft | AttachmentList;

function analyzeSubmission(input: Submission) {
  // ...your solution...
}
```

---

## Solutions

### Level 1: The Unknown Barrier
```ts
function processInput(data: unknown): void {
  if (typeof data === 'string') {
    console.log(`Text length: ${data.length}`);
  } else if (Array.isArray(data)) {
    console.log(`List size: ${data.length}`);
  }
}
```

### Level 2: The Specialized Guard
**What is a type guard?**
First of all, a typeguard is a function taking any arguments and returning a boolean. (also known as  predicate function) with a special
'return type annotation' of the form `paramName is Type`. This annotation tells the TypeScript compiler that if the function returns `true`,
then the parameter `paramName` can be treated as having the type `Type` within the scope of the "true" branch.
This allows the compiler to safely infer the type within the "true" branch and ensures type safety.

```ts
type ApiResponse = {
  data: string | string[];
  status: number;
};

// Type guard for ApiResponse
function isApiResponse(obj: unknown): obj is ApiResponse {
  if (!(
    typeof obj === 'object' &&
    obj !== null &&
    'status' in obj && typeof obj.status === 'number' &&
    'data' in obj
  )) {
    return false;
  }
  if (typeof obj.data === 'string') {
    return true;
  }
  if (!Array.isArray(obj.data)) {
    return false;
  }
  return obj.data.every((x: unknown) => typeof x === 'string');
}

function handleResponse(response: unknown) {
  if (isApiResponse(response)) {
    console.log(response.data);
  }
}

const foo = {
  data: 'bla',
  status: 0,
  more: true
};

// This passes, but it's not clear if it's intended.
isApiResponse(foo); // true

// Stricter version: checks for only the allowed keys
function strictIsApiResponse(obj: unknown): obj is ApiResponse {
  return isApiResponse(obj) && Object.keys(obj as object).length === 2;
}

strictIsApiResponse(foo); // false
```

**What does `isApiResponse` do?**

The function checks whether the given object has the structure of `ApiResponse` (i.e., the properties `data` and `status` with the correct types). It serves as a type guard for the `ApiResponse` type.

---

### Level 3: The Property Dilemma
```ts
interface TextDraft {
  content: string;
  length: number;
}

interface AttachmentList {
  files: string[];
  length: number;
}

type Submission = TextDraft | AttachmentList;

function isTextDraft(obj: unknown): obj is TextDraft {
  return (
    typeof obj === 'object' &&
    !!obj &&
    'content' in obj && typeof (obj as any).content === 'string' &&
    'length' in obj && typeof (obj as any).length === 'number'
  );
}

function analyzeSubmission(input: Submission) {
  // content -> uppercase if input is TextDraft
  if (isTextDraft(input)) {
    console.log(input.content.toUpperCase());
  }
  // show first file if input is AttachmentList
  if ('files' in input && Array.isArray((input as any).files)) {
    console.log((input as any).files[0]);
  }
}
```

**Note:**
A simple check on `.length` is not sufficient here, since both types have this property. Instead, you should check for unique properties like `content` or `files`.

---

## Additional Notes

### Generics

It is not possible to check for generic types at runtime in TypeScript, since type parameters are erased during compilation. For example:

```ts
interface Container<T> {
  data: T;
}

function isContainer<T extends string | number>(obj: unknown): obj is Container<T> {
  // This will not work as expected, since T is not available at runtime.
  return (
    typeof obj === 'object' && !!obj &&
    'data' in obj
    // typeof obj.data === T // Error: 'T' only refers to a type, but is being used as a value here
  );
}
```

### hasOwnProperty

When working with classes, it is often useful to use `hasOwnProperty` to check for properties, especially when dealing with inheritance. This helps avoid false positives from inherited properties. However, there are edge cases depending on how and where a value is assigned:

```ts
class Foo {
  maybeFoo?: boolean;
  thisIsFoo: boolean = true;
  fooFn() { return 1; }
}

class Bar extends Foo {
  bar: number = 1;
}

const f = new Foo();
const b = new Bar();

a bconsole.log('f has maybeFoo', f.hasOwnProperty('maybeFoo')); // false
console.log('f has thisIsFoo', f.hasOwnProperty('thisIsFoo')); // true
console.log('f has fooFn', f.hasOwnProperty('fooFn')); // false

console.log('b has maybeFoo', b.hasOwnProperty('maybeFoo')); // false
console.log('b has thisIsFoo', b.hasOwnProperty('thisIsFoo')); // true
console.log('b has fooFn', b.hasOwnProperty('fooFn')); // false
console.log('b has bar', b.hasOwnProperty('bar')); // true

console.log('Foo.prototype has fooFn', Object.getPrototypeOf(f).hasOwnProperty('fooFn')); // true
console.log('Bar.prototype has fooFn', Object.getPrototypeOf(b).hasOwnProperty('fooFn')); // false
```

#### The `?` Issue
This can mostly be avoided by using `null` instead of `undefined`, because `null` must be explicitly assigned.

### Functions vs Properties

Functions are assigned on the prototype, while properties are assigned in the constructor (i.e., on the instance). 
This means that `hasOwnProperty` will return `false` for functions defined in a class, as they are not own properties
of the instance but rather part of the prototype chain.  At the same time, for properties 'hasOwnProperty' will return 'true' 
even if the property is defined in the base class, as it is assigned to the instance.

If a derived class adds a method, this can be used to distinguish between a base class and a derived class, as the 
derived class will have an own property for the method, while the base class will not.
