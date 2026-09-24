# Namespaces

This chapter continues `01-types.md`. It defines how declarations are named
across files: a directory is a namespace, and `use` brings a name into scope.

A **directory** is a namespace. A **file** is not: every file in a directory
contributes to that directory's namespace, so a large namespace can be split
across files without becoming any deeper.

```text
src/
  main.xyz            → the root namespace
  std/
    meta.xyz          → std::meta
    io.xyz            → std::io
    meta/
      repr.xyz        → std::meta::repr
```

A declaration lands in the namespace of the directory its file sits in. A
subdirectory is a sub-namespace, so `std/meta/repr.xyz` holds `std::meta::repr`
while `std/meta.xyz` holds items of `std::meta` itself.

Paths are absolute and start at the root of the project. `std` is the standard
library — a reserved name at the root.

## Visibility

An item is private to its namespace unless it is marked `pub`:

```rust
pub struct TypeInfo { ... }   // visible outside std::meta
struct Helper { ... }          // visible only inside std::meta
```

Private means visible to every file of the same directory. A file is not a
namespace, so there is no file-level privacy: to hide a helper from the rest of
a namespace, give it a directory of its own.

## Use

`use` brings a name into scope:

```rust
use std::meta::TypeInfo;              // one item
use std::meta::{TypeInfo, Field};     // several
use std::meta;                        // the namespace itself — meta::TypeInfo
```

A path begins with a name that is in scope. `std` always is; `use` puts others
there — including a namespace, which is what makes `meta::TypeInfo` resolve
after `use std::meta;`. Relative paths (`self::`, `super::`) are not defined.
Two `use` of the same name in one file is a compile error.

An enum is a namespace of its variants (`09-match.md`), so a variant is
imported the same way:

```rust
use Color::Red;

let c = Red;
```

`Some` and `None` need no `use` — writing them bare is part of the `?T` sugar
(`05-traits.md`), not of this mechanism.

## Open items

- A word for the root. Paths simply start at the root for now; something
  better than Rust's `crate` may still turn up.
- Relative paths (`self::`, `super::`) — deferred until deep nesting hurts.
- Glob imports (`use std::meta::*`) and renaming (`use X as Y`).
- External libraries, and how their paths enter the root.
