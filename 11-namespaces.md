# Namespaces

This chapter continues `01-types.md`. It defines how declarations are named
across files: a directory is a namespace, and `use` brings a name into scope.

A **directory** is a namespace. A **file** is not: every file in a directory
contributes to that directory's namespace, so a large namespace can be split
across files without becoming any deeper.

```text
src/
  main.xyz            → the root namespace
  net/
    socket.xyz        → net
    tls.xyz           → net — another file, the same namespace
    pool.xyz          → net::pool
    pool/
      conn.xyz        → net::pool::conn
```

A declaration lands in the namespace of the directory its file sits in. A
subdirectory is a sub-namespace: `net/pool/conn.xyz` holds `net::pool::conn`,
while `net/pool.xyz` — the file beside the directory — holds items of
`net::pool` itself.

Paths are absolute — they start at the root of the project. `std` is the
standard library: it lives outside the project tree and appears as a name at the
root. Its name is reserved, so no namespace may have a `std` of its own. How a
name is found is in Name resolution below.

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
use std::meta::{TypeInfo, Field};     // several — instead of the line above
use std::meta;                        // the namespace itself — meta::TypeInfo
use std::meta::*;                     // every pub item — instead of the above
```

A `use` puts a name into scope — an item, a namespace, or with `*` every `pub`
item. That is what makes `meta::TypeInfo` resolve after `use std::meta;`. How a
name is found, and what happens when two collide, is in Name resolution below.

There is no renaming: `use X as Y` is not defined. A name that collides is
reached by its full path, or left out of a brace list — see below.

An enum is a namespace of its variants (`09-match.md`), so a variant is
imported the same way:

```rust
use Color::Red;

let c = Red;
```

`Some` and `None` need no `use` — writing them bare is part of the `?T` sugar
(`05-traits.md`), not of this mechanism.

## Name resolution

A name is looked up in the current namespace first — the directory the file sits
in, which holds both the declarations its files make and its sub-namespaces:

```rust
// src/net/tls.xyz — the namespace `net`
pub struct Tls { ... }

fn handshake(s: *Socket, t: *Tls) -> () { ... }
//                ^^^^^^ Socket is declared in net, and so is this file
```

If it is not there, the names `use` brought in are tried, and then the root's
sub-namespaces. `std` is one of those, so a standard-library path reads the same
from anywhere:

```rust
// src/net/pool/conn.xyz — the namespace `net::pool`
let t = std::meta::TypeInfo{ ... };
```

`::name` skips the first two and takes `name` from the root. That is how a
sub-namespace of the current one is told apart from a sub-namespace of the root:

```rust
// a file of std::meta; the project's root also has a `repr`
let a = repr::Foo;    // std::meta::repr::Foo
let b = ::repr::Foo;  // repr::Foo — the root's
```

A name brought in by `use` that the current namespace already declares is a
compile error, as are two `use` of the same name. Neither has a renaming to fall
back on; both are resolved by writing the full path instead:

```rust
// src/net/tls.xyz — the namespace `net`, which already declares a `Socket`
use quic::Socket;     // ❌ net already declares a Socket

fn g(s: *quic::Socket) -> () { }   // ✅ reach it by path
```

A glob is all or nothing — no renaming, and no way to leave one name out. If any
name it brings in collides, the whole `use` is an error, and the way out is the
brace form or the full path:

```rust
use quic::*;                  // ❌ quic has a Socket, and so does net
use quic::{Client, Server};   // ✅ the ones that do not collide
```

There are no relative paths — `self::` and `super::` are not defined, and nothing
needs them: a sibling is in the current namespace and is named with no path at
all, and everything else is named absolutely. What a path means therefore does
not depend on which file it is written in.

## Entry and artifacts

The compilation unit is the whole project. The compiler reads the tree from the
root and sees every declaration at once, which is what two rules elsewhere
already assume: an instantiation is emitted once globally no matter how many
namespaces call it, and an impl is checked at declaration against every existing
impl (`04-generics.md`). Separate compilation would break both, so v0 does not
have it; incremental builds are a matter of caching, not of the language, and
are left to the future.

A project builds one artifact. If the root namespace declares `fn main`, the
artifact is an executable; if it does not, it is a library — something another
project depends on, and nothing more. A project is one or the other, never
both, and there is no manifest and no target list.

`main` is a name, not a keyword and not an attribute: the compiler looks for
`fn main` in the root namespace, in any file of it. It may return `()`, or it
may return `E?()` for an error type of the program's choosing — the sugar that
makes `?` usable in `main` itself:

```rust
// src/main.xyz — the root namespace
fn main() -> io::Error?() {
  let f = open(config()?)?;    // ? hands errors back, main is the last stop
  ...
}
```

The compiler arranges the platform's entry — an unmangled C `main` that calls
this one — so the name never meets the mangler. How the program ends follows
from how `main` ends:

| `main` ends | the program |
| --- | --- |
| returns `()`, or `Ok` | exits with code 0 |
| returns `Err(e)` | prints `e`, exits with code 1 — a clean exit, not an abort: the error path is a normal one, and the state is trusted |
| panics | aborts (`01-types.md`, Panic) |

Printing an `Err` needs no trait: the runtime prints through reflection — the
variant name for an enum, field by field for a struct — so any error type works
without a derive.

## Tests

`#[test]` (`01-types.md`) marks a test function. A test may return `()`, or
`E?()` for an error type of its choosing — the same two shapes `main` has, for
the same reason: `?` in a test hands the error straight to the runner.

The test artifact is a second product a project can build, beside the library
or the executable: the compiler collects every `#[test]` function — from every
namespace, `pub` or not — into a table the generated runner walks:

```rust
// the compiler generates this table and a runner over it
struct TestCase {
  name: []u8,          // the function's name
  desc: []u8,          // #[test("...")] — empty when absent
  run:  fn() -> (),
}
```

The runner is the entry point of the artifact, the way the shim is for `main`:
a project's own `fn main` is not involved and need not exist. A test that
panics counts as failed — the report says so and the runner moves on — and a
test returning `Err` is failed the same way: the runner prints the error
(through the same reflection printing `main`'s `Err` uses) and continues. The
artifact exits 0 when every test passed, 1 otherwise.

`#[build]` applies as usual — a `#[build(debug)]` helper compiled out in
`release` is absent from a test build too, which is built in `debug` shape.
There is no `test` build mode: the artifact is its own product, and the
debug/release axis is not what distinguishes it.

## Open items

- External libraries, and how their paths enter the root. They will distribute
  as source — the compiler has to read a library to check against it — but how
  a dependency is declared, and what happens when two want different versions
  of the same library, waits for a manifest to exist.
