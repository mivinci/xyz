# Traits

This chapter continues `04-generics.md`.

## Trait

A trait is a set of function signatures. A type implements the trait by
providing them:

```rust
trait Show {
  fn show(self: *Self) -> ();
}

impl Show for Point {
  fn show(self: *Self) -> () {
    print(@cast<voidptr>(self));
  }
}
```

`print` and `close` below are ordinary functions from `std::io`, brought into
scope with `use std::io;` (`11-namespaces.md`) — not `@` builtins, which are
listed in `08-reflection.md`.

`self` is an ordinary parameter — `*Self` for a read-only method, `*mut Self`
for a mutating one.

A method call is sugar: the receiver is adapted to the `self` the method
declares, so `p.len()` is `Point::len(&p)` and `it.next()` is `It::next(&mut it)`.
Taking `&mut` still requires a `mut` slot, exactly as `&mut` does anywhere else
(`01-types.md`). Where the receiver is a pointer it is dereferenced first, so
`sp.len()` with `sp: *Point` is `Point::len(&*sp)` — which is `sp` again. A
`dyn A` handle is passed the same way (`06-dispatch.md`). The explicit form
stays available — `Show::show(&p)` is the same call written out.

Everywhere else there is no implicit borrowing: an argument list passes exactly
what you wrote (`03-move.md`).

`Self` refers to the implementing type. Inside an `impl`, it is a synonym for
the name in `for`.

## Associated Items

A trait declares associated types and constants, which an impl supplies
alongside the functions:

```rust
trait Iterator {
  type Item;                          // associated type
  fn next(self: *mut Self) -> ?Self::Item;
}

impl<T> Iterator for []mut T {
  type Item = T;                      // impl supplies the type
  fn next(self: *mut Self) -> ?T { ... }
}
```

`Self::Item` names it inside the trait; outside, `Iterator::Item` names it for
the trait as a whole, and `It::Item` for a type `It` that implements it.

The same `type` keyword names a type at the top level — a transparent type
alias, generic or not (`01-types.md`).

A trait can also declare an associated constant, read with `::` and supplied
by the impl. A struct or enum supplies its associated constants and methods
through an inherent impl — the next section.

## Inherent Impl

An `impl` without a trait name attaches methods and associated constants
directly to a type. No trait needs to be in scope to call them:

```rust
impl Point {
  fn len(self: *Self) -> usize { ... }
}

let p = Point{ x: 0, y: 0 };
p.len();            // inherent method — no trait import
```

The target is a shape pattern, exactly as in a struct declaration
(`04-generics.md`). A specialized struct gets one inherent impl per
specialization, each target repeating the same pattern:

```rust
// in std::meta — the predicate struct (04-generics.md)
struct is_same<A, B> {}          // shape only
struct is_same<T, T> {}

impl<A, B> is_same<A, B> { const value: bool = false; }
impl<T>    is_same<T, T> { const value: bool = true;  }

#assert(is_same<i32, i32>::value);
#assert(!is_same<i32, u32>::value);
```

An inherent impl supplies associated constants and methods, but not fields —
fields live in the struct declaration. The `(T, T)` repeated-variable pattern
makes the two impls target disjoint instantiations, so they do not conflict.

An associated constant lives in the type, not the instance: reading it takes
no space — `@sizeof<is_same<i32, i32>>()` is `0`.

Inherent impls and trait impls are separate namespaces: `p.len()` calls the
inherent method, `Show::show(&p)` the trait one. The specificity and
disjointness rules that order trait impls (`04-generics.md`) apply to
inherent impls of a specialized struct unchanged.

## Generics

How generic functions are instantiated, how bounds are checked, and how
multiple impls of a trait are ordered — see `04-generics.md`.

## Operators

`+`, `<` and `==` are trait methods as well — `Add`, `Ord` and `Eq` — and an
operator is sugar for the call. See `07-operators.md`.

## Copy

`Copy` is a marker trait — it has no functions, so its impl is empty. The
compiler accepts it only when every field (or element) is itself `Copy` and no
destructor exists (see `Drop` below).

```rust
struct Point {
  x: u32,
  y: u32,
}

impl Copy for Point { }
```

See `03-move.md` for what `Copy` does on assignment.

## Drop

`Drop` has a single function, `drop`, which receives the value by ownership.
It runs when the binding that owns the value reaches the end of its scope —
see `03-move.md` for the timing rules.

```rust
impl Drop for File {
  fn drop(mut self: Self) -> () {
    close(self.fd);
  }
}
```

`mut self: Self` — a parameter is a slot like any other (`01-types.md`), so
`mut` marks it writable; taking `Self` by value is what transfers ownership.
The body must treat the zero value as a no-op — this is what makes `@take` sound, and it
forbids types whose zero value is a live resource:

```rust
impl Drop for BadFd {
  fn drop(mut self: Self) -> () {
    close(self.fd);   // ❌ if zero means fd 0, closing stdin
  }
}
```

`Copy` and `Drop` are mutually exclusive. The compiler rejects `impl Copy`
when any field is not `Copy` or a destructor exists.

## Option

`?T` is sugar for `Option<T>`, a regular enum in the standard library:

```rust
enum Option<T> {
  None,
  Some(T),
}

// Some(3): ?u32
```

`Some` and `None` are written without a prefix — that is part of the `?T` sugar,
not of `use` (`11-namespaces.md`).

Niche optimization is a compiler specialization for `Option` specifically,
not a trait-system feature: when `T` has an unused bit pattern, `@sizeof(?T)`
equals `@sizeof(T)`; otherwise `?T` grows by a tag. See `04-generics.md` for the
specialization rules.

## Coherence

At most one `impl` may exist for a given trait and concrete type, globally —
generic impls with disjoint or strictly ordered bounds are the exception, see
`04-generics.md`. To keep two libraries from colliding, an `impl` is rejected
unless the trait or the type was defined in the current namespace — the orphan
rule:

```rust
impl Show for u32 { ... }   // ❌ neither Show nor u32 is ours
```
