# xyz

A system programming language, still being designed. This is its
**specification draft** — not a tutorial, and not an implementation.

`xyz` is the working codename for the project; the language has no final name
yet. The `.xyz` file extension used throughout `09-namespaces.md` is a
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
| [06-reflection.md](./06-reflection.md) | compile-time execution, `TypeInfo`, the builtin table |
| [07-match.md](./07-match.md) | pattern matching |
| [08-iteration.md](./08-iteration.md) | `Iterator`, `while`, `for`, `if`, `return` |
| [09-namespaces.md](./09-namespaces.md) | a directory is a namespace, `use` |
| [10-macros.md](./10-macros.md) | why there is no user-defined macro system |

## Notation

- `@xxx` is a builtin function and `#xxx` is a built-in construct — the full
  list is in `06-reflection.md`
- `mut` marks **the slot right after it**: a variable, a field, an element, or a
  parameter. There is one rule, not one per position.
- ✅ and ❌ in an example mean it does or does not compile

## Status

A design in progress. The specification is internally consistent at the moment;
`REVIEW.md` records what has been reviewed and how each point was settled.

Still open:

- Whether a macro system should exist at all — `10-macros.md` currently argues
  against one, but the conclusion is deliberately left open
- The open items at the end of `09-namespaces.md`: a name for the root,
  relative paths, glob imports, external libraries
