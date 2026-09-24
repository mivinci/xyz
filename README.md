# xyz

A system programming language, still being designed. This is its
**specification draft** — not a tutorial, and not an implementation.

`xyz` is the working codename for the project; the language has no final name
yet. The `.xyz` file extension used throughout `11-namespaces.md` is a
placeholder for the same reason.

## Chapters

Read them in order; each chapter opens by naming the one it continues.

| file | what it defines |
| --- | --- |
| [01-types.md](./01-types.md) | types and `mut`, arrays, slices, structs, pointers, enums, strings |
| [02-layout.md](./02-layout.md) | size and alignment, `#repr` |
| [03-move.md](./03-move.md) | move semantics, `Copy`, `Drop`, `@take` |
| [04-generics.md](./04-generics.md) | generics, specialization, shape patterns, variadics |
| [05-traits.md](./05-traits.md) | traits, associated items, inherent impls, `Option` |
| [06-dispatch.md](./06-dispatch.md) | `dyn A`, dynamic dispatch, object safety |
| [07-operators.md](./07-operators.md) | operators as trait methods, `Add`/`Ord`/`Eq` |
| [08-reflection.md](./08-reflection.md) | compile-time execution, `TypeInfo`, the builtin table |
| [09-match.md](./09-match.md) | pattern matching |
| [10-iteration.md](./10-iteration.md) | `Iterator`, `while`, `for`, `if`, `return` |
| [11-namespaces.md](./11-namespaces.md) | a directory is a namespace, `use` |
| [12-macros.md](./12-macros.md) | the case against a user-defined macro system (not settled) |

## Notation

- `@xxx` is a builtin function and `#xxx` is a built-in construct — the full
  list is in `08-reflection.md`
- `mut` marks **the slot right after it**: a variable, a field, an element, or a
  parameter. There is one rule, not one per position.
- ✅ and ❌ in an example mean it does or does not compile

## What xyz guarantees

There is no borrow checker and no reference type — one pointer family, and
lifetimes nowhere — so the guarantees are uneven on purpose:

- **Guaranteed statically.** No use after move, no double free from a move, no
  write through a shared pointer, no move out of a borrowed place, no dropping
  through a pointer that is not known to be exclusive.
- **Not proven.** Exclusivity and non-dangling hold only where the compiler can
  see them — within a function. Across a function boundary there is no lifetime
  information, so a dangling use or an aliasing violation is undefined
  behaviour. A `debug` build may catch some of these; the language does not
  define a mechanism, and promises nothing. This is the same bargain a slice
  makes (`01-types.md`).
- **Deliberately unchecked.** A `static mut` is the one place aliasing is
  allowed without any check: any function may write it, and nothing proves only
  one does. Statics are also never destructed (`01-types.md`).
- **Not addressed.** Data races, iterator invalidation, and anything else that
  would need aliasing to be tracked across the whole program.

The shape is Rust's ownership — moves, `Copy`, `Drop`, destructors inserted
statically — with Zig's answer to aliasing, plus a few things neither has: `mut`
as part of the type, specialization ordered by shape pattern and checked where
the impls are declared, and compile-time reflection in place of a macro system.

## Status

A design in progress. The specification is internally consistent at the moment;
`review/` holds one file per review, named `review-YYYYMMDD-NN.md`.

Still open:

- Whether a macro system should exist at all — `12-macros.md` currently argues
  against one, but the conclusion is deliberately left open
- The open items at the end of `11-namespaces.md`: a name for the root,
  relative paths, glob imports, external libraries
