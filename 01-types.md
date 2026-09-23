# Types

This chapter requires that you've already known that `@xxx` is a builtin function and `#xxx` is a macro.

## Primitives

```rust
i8, i16, i32, i64, i128
u8, u16, u32, u64, u128
isize, usize
bool
f32, f64
voidptr
```

There is no `char` type: a character literal such as `'a'` or `'\0'` is just a
`u8` literal. An unsuffixed integer literal defaults to `i32` and an unsuffixed
float literal defaults to `f32`, so `3.14` is an `f32`.

## Mutability

`mut` marks whether **the slot right after it** can be written:

- `let mut a` — the slot is the variable itself, i.e. whole-value assignment
- `mut c: f32` — the slot is the field, i.e. field assignment
- `[3]mut u32`, `(u32, mut f32)` — the slot is the element; elements have no
  name, so `mut` goes before the type

The two levels are orthogonal:

| declaration                | `a = v` (whole value) | `a[0] = v` / `a.b = v` (interior) |
| -------------------------- | --------------------- | --------------------------------- |
| `let a = [3]u32{}`         | ❌                    | ❌                                |
| `let mut a = [3]u32{}`     | ✅                    | ❌                                |
| `let a = [3]mut u32{}`     | ❌                    | ✅                                |
| `let mut a = [3]mut u32{}` | ✅                    | ✅                                |

Immutability is therefore **not** deep: an immutable binding can still be
mutated through its `mut` fields.

```rust
struct P {
      a: u32,
  mut b: u32,
}

let p = P{ a: 1, b: 2 };

p = P{ a: 3, b: 4 };  // ❌ the binding p is not mut
p.a = 3;              // ❌ P::a is not mut
p.b = 4;              // ✅ P::b is mut
```

`mut` written inside a type is part of that type: `[3]mut u32` and `[3]u32` are
two distinct types, and so are two structs differing only in the `mut` of a
field. A value of a mutable type can be used where the immutable one is
expected — dropping writability is always safe — but not the other way around.
For example, `[3]mut u32` is usable wherever `[3]u32` is expected. A move is the
one exception: a value may be moved into a slot of the mutable type, because
that slot is new and nothing aliases it yet — raising writability that no one can
observe is as safe as dropping it. `let a: [3]mut u32 = b;` therefore accepts a
`b: [3]u32`. Since the `mut` is part of the type, the two types compare
distinct: `is_same<[3]mut u32, [3]u32>::value` is false.

`mut` is only meaningful inside a compound type — `*mut T`, `[3]mut T`,
`mut c: T` — or on a named slot: `let mut a` for a variable, `mut self: Self`
for a parameter. There is no standalone `mut i32`.

## Array

immutable

```rust
let a = [3]u32{};         // [0, 0, 0]
let b = [3]u32{1};        // [1, 0, 0]
let c = [5]u32{1, 2, 3};  // [1, 2, 3, 0, 0]

a[0] = 1;      // ❌ the elements are not mut
a = [3]u32{};  // ❌ the binding a is not mut
```

Giving more initializers than the length is a compile error:

```rust
let d = [2]u32{1, 2, 3};   // ❌ too many initializers
```

mutable

```rust
let a = [3]mut u32{1, 2, 3};

a[0] = 42;

#assert(a[0] == 42);
#assert(a[1] == 2);
#assert(a[2] == 3);
```

## Slice

`[]T` is a fat pointer: a pointer to the first element plus a length. It borrows
elements owned by something else, so it never allocates. The two parts are
named — `s.ptr` is `*T` (`*mut T` for `[]mut T`) and `s.len` is a `usize`. They
behave like `mut` struct fields, so both can be written; the way to advance the
view, though, is to slice it — `s = s[1..]` below. Neither touches the elements
the slice borrows, so a `[]T` whose elements cannot be written may still be
advanced. Writing the slice itself — `s = ...` — is governed by the binding's
`mut`, exactly as `p = ...` is.

```rust
let a = [3]u32{1, 2, 3};
let s: []u32 = a;   // [3]u32 → []u32

#assert(s[0] == 1);
#assert(s[2] == 3);
```

As with arrays, `mut` marks whether the elements can be written:

```rust
let a = [3]mut u32{1, 2, 3};
let s: []mut u32 = a;

s[0] = 9;   // ✅ the elements are mut
```

Slicing a slice moves its start — `s[a..b]` is the elements from `a` up to `b`,
and `s[a..]` reaches the end. Nothing is copied; the result borrows the same
elements:

```rust
let rest = s[1..];   // []u32 — the same elements, one shorter
```

A range that leaves the slice is caught by the runtime checks in `debug` and is
undefined behaviour in `release`, exactly as an out-of-range index is. The same
`[a..b]` works on a tuple (`04-generics.md`).

There is no borrow checker. A slice is rejected at compile time only when the
compiler can see that it outlives what it borrows — returning a slice of a local
array, say. Across function boundaries there is no lifetime information, so a
dangling slice falls to the runtime checks in `debug` mode and is undefined
behaviour in `release`.

An array never decays to a plain pointer. To get a `*T`, take the address of an
element explicitly:

```rust
let p: *u32 = &a[0];
```

## Struct

```rust
struct X {
      a: u32,
      b: u32,
  mut c: f32,
}
```

where c is mut

```rust
let a: X = {};  // filled with zero values
let b: X = { a: 1, b: 42, c: 3.14 };

a.b = 2;     // ❌ X::b is not mut
b.c = 2.71;  // ✅ X::c is mut

#assert(a.a == 0);
#assert(a.b == 0);
#assert(a.c == 0.0);
```

or

```rust
let a = X{};  // filled with zero values
let b = X{ a: 1, b: 42, c: 3.14 };
```

## Type Alias

`type` gives a type a shorter name. An alias is transparent — it *is* the type
it names, not a new one — so `is_same` sees through it:

```rust
type MyInt = i32;

#assert(is_same<MyInt, i32>::value);
```

Because it is transparent, the alias and its target are one type — there is no
way to tell them apart, so an alias cannot take an `impl` of its own
(`05-traits.md`).

An alias may be generic. Instantiation substitutes the arguments into the
target, exactly as if the target were written out:

```rust
type Vec<T>   = []T;
type Pair<A, B> = (A, B);

#assert(is_same<Vec<u32>, []u32>::value);
#assert(is_same<Pair<i32, u8>, (i32, u8)>::value);
```

An alias is a name, not a shape pattern: it has no specialization, and it may
not be recursive — `type A = B; type B = A;` never terminates, so cycles are a
compile error.

An alias may be declared in any block, not only at the top level. Its scope is
the block, like a `let`. A local alias follows the same rules as a top-level
one — transparent, generic or not, and able to name the enclosing function's
type parameters:

```rust
fn collect<T>(xs: []T) {
  type MyVec      = []T;       // captures the function's T
  type Wrapper<X> = (X, X);    // declares its own parameter X
  let v: MyVec = xs;
  let w: Wrapper<i32> = (1, 2);
}
```

`MyVec` is `[]T`, `Wrapper<i32>` is `(i32, i32)` — neither is a new type. A
local alias declares its parameters exactly as a function does: `<X>` shadows
an outer `X`, while a bare `T` refers to the enclosing function's `T`.

## Tuple

```rust
let a: (u32, mut f32) = (42, 3.14);

a.0 = 1;     // ❌ the 1st element is not mut
a.1 = 2.71;  // ✅ the 2nd element is mut

#assert(a.0 == 42);
#assert(a.1 == 2.71);
```

where the 2nd element is mut

The empty tuple `()` is the unit type — it holds no value, and a function
without a return value returns `()`. There is no `void`, so generic code never
needs a `FixVoid`:

```rust
#assert(@sizeof(()) == 0);
```

There is no `never` type: a function that never returns still returns `()`, and
divergence is not tracked in the type system.

The `...` operator in expression position expands a tuple into individual
values — `f(t)` passes the tuple as one argument, `f(...t)` passes each element
as a separate argument:

```rust
let t = (1, 2, 3);

f(t);    // f receives one argument of type (i32, i32, i32)
f(...t); // f receives three arguments: 1, 2, 3
```

It applies to any tuple, not just packs. Slices expand the same way. The
type-position use of `...` — declaring a pack — is defined in `04-generics.md`.

## Union

```rust
union X {
  a: u16,
  b: u32,
}
```

`@sizeof`, `@alignof` and `@offset` are all in **bytes**.

When no field is given, the whole union is zeroed, so every field reads as its
zero value:

```rust
let a = X{};

#assert(@sizeof(a) == 4);
#assert(@sizeof<X>() == 4);
#assert(@alignof(a) == 4);
#assert(@alignof<X>() == 4);
#assert(@offset<X>("a") == 0);
#assert(@offset<X>("b") == 0);
```

Once a field has been written, reading a different one behaves exactly as it does
for a union in C — the value is unspecified, so don't rely on it.

Fields of a union cannot be marked `mut`, since writing any field overwrites the
memory shared by all the others:

```rust
union Y {
      a: u16,
  mut b: u32,   // ❌ a union field cannot be mut
}
```

Instead, mutability is expressed on the slot holding the union, which makes the
whole union writable:

```rust
let mut x = X{ b: 42 };

x.b = 7;   // ✅ x is mut, so the whole union is writable
```

Taking a mutable pointer to a field follows the same rule: `&mut x.b` needs no
`mut` on `b` — a writable union is writable through every one of its fields.
That is what lets `@take(&mut x.b)` retrieve a value out of a union
(`03-move.md`).

## Pointer

`*T` points to a value that cannot be written through the pointer, and `*mut T`
points to one that can. Both are nonnull. `?` makes a pointer nullable: `?*T`,
`?*mut T`.

```rust
let a = 42;
let p: *i32 = &a;          // ✅ a is not mut, so only *i32 is available

let mut b = 42;
let q: *mut i32 = &mut b;  // ✅ b is mut

*q = 7;

#assert(*q == 7);
#assert(*p == 42);
```

`&mut` requires the slot itself to be `mut`:

```rust
let c = 42;
let r: *mut i32 = &mut c;  // ❌ the binding c is not mut
```

`*p` reads the pointee and `*p = v` writes it. A pointer is dereferenced as far
as it needs to be to reach a member, so `sp.b` is `(*sp).b` — there is no `->` —
and a method is called the same way (`05-traits.md`):

```rust
let mut s = P{ a: 1, b: 2 };
let sp: *mut P = &mut s;

sp.b = 3;   // ✅ P::b is mut
sp.a = 3;   // ❌ P::a is not mut
```

A pointer can be walked, but a slice is usually the better tool: `s[1..]` moves
a whole view at once and cannot leave the sequence (Slice above) — the slice
iterators in `10-iteration.md` are written that way. Arithmetic is for what a
slice cannot express, because there is no length to carry: an allocator walking
a block, or a walk that knows only where it ends:

```rust
let next: *i32 = p + 1;   // steps by @sizeof(i32)
```

`+` and `-` take an integer; there is no subtraction of two pointers. `voidptr`
cannot be walked, since it has no element size.

Pointers of the same type compare with `<`, `>`, `<=` and `>=`, which is what a
walk needs in order to know where to stop:

```rust
let mut p: *i32 = &a[0];
let end: *i32 = p + 3;

while p < end {
  use(*p);
  p = p + 1;
}
```

The order is unspecified unless both point into the same array — as in C.
Equality, `==` and `!=`, works on any two pointers of the same type; it is
already what a `?*T` check compares against `None`.

Holding a `*T` and a `*mut T` to the same value at the same time is allowed,
exactly as in C — an immutable pointer does not promise that the value stays
unchanged.

`@sizeof(*T)` and `@sizeof(*mut T)` are equal, and returning the address of a
local is a compile error.

### Nullability

`?` is a general modifier: `?T` is "either a `T` or nothing", so `?u32` and
`?*mut i32` are both valid. A nullable pointer must be checked before it is used;
after an explicit check it narrows to the nonnull type:

```rust
let p: ?*mut i32 = &mut b;

if p != None {
  *p = 7;   // ✅ p is *mut i32 here
}

*p = 8;     // ❌ outside the check, p is ?*mut i32 again
```

Types distinguish them: `is_same<?*i32, *i32>::value` is false.

`?T` is sugar for `Option<T>`. When `T` has an unused bit pattern the compiler
uses it as a niche, so `@sizeof(?*T)` equals `@sizeof(*T)` — the null pointer
value represents the empty case. Types without a spare bit pattern carry a tag
instead, so `?u32` is larger than `u32`.

`voidptr` is the opaque pointer type. It can neither be dereferenced nor walked
— it has no element size — so `@cast` it to a concrete pointer type first:

```rust
let v: voidptr = @cast<voidptr>(q);
let back: *mut i32 = @cast<*mut i32>(v);
let bad = v + 1;   // ❌ voidptr cannot be walked
```

## Cast

`@cast<T>(a)` performs a well-defined conversion, like `static_cast` in C++:

```rust
let a = @cast<f64>(1);      // i32 → f64
let b = @cast<u32>(3.14);   // f32 → u32, truncated to 3
```

## Enum

```rust
enum X(u32) {
  A = 0,
  B,
  C,

  D = 10,
  E,
}
```

The variants are of type `X`, not of the underlying type, so `@cast` is needed
to get the integer out:

```rust
#assert(@cast<u32>(X::A) == 0);
#assert(@cast<u32>(X::B) == 1);
#assert(@cast<u32>(X::C) == 2);
#assert(@cast<u32>(X::D) == 10);
#assert(@cast<u32>(X::E) == 11);
```

## String

### String Literals

A string literal is a `[]u8` — a slice into static read-only memory. Like in
C it has a fixed address, so it is a slice, not an array value:

```rust
let a = "hi";   // a: []u8

a[1] = 'a';   // ❌ the elements of []u8 are not mut
```

### C String

A `c` prefix makes a NUL-terminated slice — the C string literal, for FFI:

```rust
let a = c"hi";   // a: []u8

#assert(a[2] == '\0');
```

### Raw String

An `r` prefix disables `\` escapes, for regexes and file paths:

```rust
let a = r"hi\nike";   // a: []u8 — backslash and `n`, not a newline
```

The prefixes combine: `cr"..."` is raw and NUL-terminated:

```rust
let d = cr"hi\n";   // d: []u8 — backslash and `n`, then '\0'
```

A raw literal cannot contain `"` — it is the delimiter. Write a quote as an
escaped `\"` in a normal literal instead; there is no `r#"..."#` nesting.

## Compile-Time Checks

Whatever the compiler can prove wrong is rejected at compile time instead of
being left to run time:

- indexing an array or a slice with a constant index that is out of range
- converting a constant to a type that cannot hold it
- arithmetic on constants that overflows
- dereferencing a `?*T` that has not been checked
- taking `&mut` of a slot that is not `mut`
- returning the address of a local
- returning a slice that outlives what it borrows
- giving more initializers than the array length

```rust
let a = [3]u32{1, 2, 3};

let x = a[3];      // ❌ index out of range, known at compile time
let y = a[1];      // ✅

let b: u8 = 300;   // ❌ 300 does not fit in u8
let c: u8 = 3;     // ✅ the constant fits, so it is converted
let d: i32 = 2147483647 + 1;   // ❌ overflow, known at compile time
```

A constant that fits the target type is converted implicitly; one that does not
requires an explicit `@cast`.

### Build Modes

Compile-time checks are always on, in every mode. Runtime checks depend on the
build mode:

- `debug` — runtime checks are inserted; an out-of-range index, an arithmetic
  overflow or a failed conversion traps
- `release` — no runtime checks; those same cases are undefined behaviour

A program therefore has to be validated in `debug` before it can be trusted in
`release`. What is a compile error stays a compile error in both modes.
