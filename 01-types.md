# Level 1: Basic Payloads

Implement the following two types/interfaces based on BackendUser:

CreateUserPayload: Requires email, firstName, and lastName.

PatchUserPayloadBasic: Allows optional updates for email, firstName, lastName, role, and isActive.

Constraint: Explicitly exclude id and server-managed timestamps (createdAt, updatedAt).

Bonus Question: In the context of API payloads, what are the pros and cons of using an interface versus a type?

# Level 2: Strict Patching (The "At Least One" Rule)
Implement a utility helper called AtLeastOne<T>.
Update your PatchUserPayload using this helper to ensure that an empty object `{}` results in a TypeScript error.

## Test Cases:

✅ `{ email: "dev@example.com" }` — Success

✅ `{ role: "editor" }` — Success

❌ `{}` — Error: Object must not be empty.

❌ `{ id: "123" }` — Error: id property is not allowed.

# Level 3: Generic Payload Utilities
Create reusable generic utilities that can be applied to any entity (User, Product, Order, etc.).

* `CreatePayload<T, KRequired>` with `T` The base Entity type and `KRequired` The keys of T that are mandatory for creation.

Logic: Mandatory keys remain required; other keys from the model become optional; id is always excluded.

* `PatchPayload<T, KPatch>` with `T` The base Entity type and `KPatch` The keys of T that are allowed to be updated.

Logic: All keys in KPatch are optional, but at least one must be provided. id must be excluded.

## Test Cases (Example with Product):

```
type Product = { id: number; sku: string; name: string; price: number; stock: number };
```

### CreatePayload Test
✅ `{ sku: "A1", name: "Gadget" }`
✅ `{ sku: "A1", name: "Gadget", price: 10 }`
❌ `{ name: "Gadget" }` (Missing mandatory 'sku')
❌ `{ id: 1, sku: "A1", name: "Gadget" }` ('id' is forbidden)

### PatchPayload Test
✅ `{ name: "New Name" }`
✅ `{ price: 19.99 }`
❌ {} (Empty object not allowed)
❌ `{ stock: 5 } (If 'stock' was not passed in KPatch)
❌ `{ id: 2 } ('id' is forbidden)


# Solution
```
interface BackendUser  {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  role: "admin" | "editor" | "viewer";
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
};

/*

CreateUserPayload: benötigt email, firstName, lastName.
PatchUserPayloadBasic: erlaubt optionale Updates (email, firstName, lastName, role, isActive).
id (und idealerweise Serverfelder wie Timestamps) wird ausgeschlossen.

*/

type CreateUserPayload = Pick<BackendUser, 'email' | 'firstName' | 'lastName'>;

type PatchUserPayloadBasic = Partial<
  Omit<BackendUser, 'id' | 'createdAt' | 'updatedAt'>
>

const payload: PatchUserPayloadBasic = {}



/*
Implementiere nur einen simplen Helper: AtLeastOne<T>.
Nutze ihn für PatchUserPayload, damit {} nicht erlaubt ist.
*/

type AtLeastOne<T, K extends keyof T = keyof T> = 
 { [P in K]-?: Required<Pick<T, P>> }[K]

type ImprovedPatchUserPayloadBasic = AtLeastOne<BackendUser>;


const patch1: ImprovedPatchUserPayloadBasic = {}

const patch2: ImprovedPatchUserPayloadBasic = {
  email: 'some@mail'
}

const patch3: ImprovedPatchUserPayloadBasic = {
  firstName: 'John',
  lastName: 'Doe'
}


type AtLeastOneAlternativeSolution<T extends object> = {
 [K in keyof T]-?: Required<Pick<T, K>> & Partial<Omit<T, K>>;
}[keyof T];

type PatchUserPayload = AtLeastOne<PatchUserPayloadBasic>;



type AtLeastOneExtended<T, K extends keyof T = keyof T> =
  Partial<Omit<T, K>> &
  { [P in K]-?: Required<Pick<T, P>> }[K]; // One of these is required


  const patch4: AtLeastOneExtended<BackendUser, 'role' | 'isActive'> = {
  firstName: 'jane'
}

const patch5: AtLeastOneExtended<BackendUser, 'role' | 'isActive'> = {
  role: 'admin',
  firstName: 'jane'
}


// Baue eine generische Utility für mehrere Entities, aber ohne zu tiefe Typmagie:         

type CreatePayload<T extends Object, Required extends keyof T>  = Pick<T, Required>;
type CreateUserPayloadGeneric = CreatePayload<BackendUser, 'email' | 'firstName' | 'lastName'>;


type PatchPayload<T, Optional extends keyof T> = AtLeastOne<T, Optional>
type PatchUserPayloadGeneric = PatchPayload<BackendUser, 'email' | 'firstName' | 'lastName' | 'role' | 'isActive'>


type Strange = CreatePayload<boolean, never>
```
