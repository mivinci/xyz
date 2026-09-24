# Preliminaries

How to read this specification, and the terms every chapter uses.

## Notation

- `@xxx` is a builtin — a function the language provides, not one written in it.
  There are ten, listed in `08-reflection.md`.
- `#xxx` is a built-in construct. There are two, listed in `12-macros.md`.
- In an example, ✅ means it compiles and ❌ means it does not.

## Terms

- A **slot** is somewhere a value can sit: a variable, a field, an element of an
  array or tuple, or a parameter. `mut` marks the slot right after it
  (`01-types.md`).
- A **place** is an expression that names a slot — `x`, `x.f`, `a[0]`, `*p`
  (`01-types.md`).
- A value is **compile-time known** when the compiler can evaluate it before the
  program runs (`08-reflection.md`).
