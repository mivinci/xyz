# Projects

This chapter continues `11-namespaces.md`. It defines what the compiler takes
a project to be: one compilation unit, one artifact, and the name that tells
an executable from a library.

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

## Open items

- External libraries, and how their paths enter the root. They will distribute
  as source — the compiler has to read a library to check against it — but how
  a dependency is declared, and what happens when two want different versions
  of the same library, waits for a manifest to exist.
