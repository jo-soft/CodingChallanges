# Challenge 4: Angular Anti-Patterns

## Task Description

### Task 1: The `effect()` Misuse

Identify the anti-pattern in the following component and propose a clean solution.

```typescript
@Component({
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    @if (userForm) {
      <form [formGroup]="userForm">
        <input formControlName="name" placeholder="Name">
        <input formControlName="email" placeholder="Email">
      </form>
    }
  `
})
export class UserEditComponent {
  userData = input.required<UserData>();
  userForm!: FormGroup;

  constructor() {
    effect(() => {
      const data = this.userData();
      this.userForm = new FormGroup({
        name: new FormControl(data.name),
        email: new FormControl(data.email)
      });
    });
  }
}
```

- What is the anti-pattern here?
- What problems does it cause?
- How does a clean solution look?

---

### Task 2: Signal Mutation & Encapsulation

Identify the anti-pattern (and the hidden bug) in the following code and propose a clean solution.

```typescript
@Injectable({ providedIn: 'root' })
export class FavoriteService {
  public favorites = signal<Product[]>([]);
}

@Component({ /* ... */ })
export class FavComponent {
  service = inject(FavoriteService);

  async addToFavs(item: Product) {
    const current = this.service.favorites();
    await this.checkPermission().then(hasAccess => {
      if (hasAccess) {
        current.push(item);
        this.service.favorites.set(current);
      }
    });
  }
}
```

- What is the anti-pattern here?
- There is an additional hidden bug — what is it and where?
- How does a clean solution look?

---

## Solutions

### Task 1: The `effect()` Misuse

**What is the anti-pattern?**

`effect()` is used to create a **brand new `FormGroup` instance** on every change of `userData`, reassigning `this.userForm` each time.

This is problematic for several reasons:

1. **Misuse of `effect()`**: `effect()` is intended for side effects (e.g. logging, DOM manipulation), **not** for creating or storing state that is bound to the template.
2. **Re-creation instead of update**: Every input change destroys the entire `FormGroup` and rebuilds it — including loss of dirty state, touched state, and validation errors.
3. **Unwanted scrolling & focus loss**: Since `userForm` is completely replaced on every change, Angular re-renders the entire form in the template. This destroys and recreates DOM elements, which can cause unwanted scroll resets and loss of input focus (hence the `@if (userForm)` workaround in the template).
4. **Mutable field**: `userForm` is reassigned from within an effect, making the data flow hard to follow and error-prone.

**Clean Solution**

Instead of recreating the `FormGroup`, initialize it **once** and only update its value on input changes via `setValue()`. Use `TypedForm<T>` for type safety.

```typescript
type TypedForm<T> = {
  [K in keyof T]: FormControl<T[K] | null>;
};

interface UserData {
  name: string;
  email: string;
}

@Component({
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="userForm">
      <input formControlName="name" placeholder="Name">
      <input formControlName="email" placeholder="Email">
    </form>
  `
})
export class UserEditComponent {
  userData = input.required<UserData>();

  userForm = new FormGroup<TypedForm<UserData>>({
    name: new FormControl(null),
    email: new FormControl(null)
  });

  constructor() {
    effect(() => {
      this.userForm.setValue(this.userData());
    });
  }
}
```

**Improvements:**
- The `FormGroup` is created **once** declaratively — no re-initialization.
- `effect()` only calls `setValue()` — a true, minimal side effect.
- The `@if (userForm)` guard in the template is no longer needed since `userForm` is never `undefined`.
- `TypedForm<UserData>` makes the form **type-safe**.

---

### Task 2: Signal Mutation & Encapsulation

**What is the anti-pattern?**

1. **Direct mutation of the signal array**: `current.push(item)` mutates the array **in-place**. Signal arrays should be treated as immutable — Angular relies on reference changes to detect updates reliably.
2. **Public `WritableSignal`**: Any component can call `favorites.set(...)` or `favorites.update(...)` directly, breaking encapsulation and removing the service's control over its own state.

**Hidden Bug**

`const current = this.service.favorites()` reads the signal value **before the `await`**. If another update changes the signal between the start of `checkPermission()` and the `.then()` callback, `current` is already **stale** — the subsequent `push` and `set` will overwrite newer state with an outdated snapshot (read-write race condition).

**Clean Solution**

```typescript
@Injectable({ providedIn: 'root' })
export class FavoriteService {
  // Internal WritableSignal — only the service may write
  private internalFavorites = signal<Product[]>([]);

  // Exposed as readonly to the outside
  public favorites: Signal<ReadonlyArray<Product>> = this.internalFavorites.asReadonly();

  public addFavorite(item: Product) {
    // update() reads and writes atomically — no race condition
    this.internalFavorites.update(current => [...current, item]);
  }
}

@Component({ /* ... */ })
export class FavComponent {
  service = inject(FavoriteService);

  async addToFavs(item: Product) {
    const hasAccess = await this.checkPermission();
    if (hasAccess) {
      this.service.addFavorite(item);
    }
  }
}
```

**Improvements:**
- **Encapsulation**: `internalFavorites` is `private` — only the service itself mutates the state.
- **Readonly outside**: `favorites` is `Signal<ReadonlyArray<Product>>` — consumers cannot write to it.
- **Immutable update**: `update(current => [...current, item])` always creates a new array — no in-place mutation.
- **Consistent `async/await`**: No `.then()` mix — cleaner, more readable control flow.
- **Race condition resolved**: Read and write logic lives entirely inside `update()` in the service, which always reads the current state at the time of the call.
