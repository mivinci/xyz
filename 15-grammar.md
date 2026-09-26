# Grammar

This chapter states the language's grammar in EBNF. It arrives in three
passes: this pass fixes the lexical grammar — the tokens everything later
parses. The second pass fixes expressions (precedence and the ambiguous
corners), the third declarations and statements, closing the language.

## Notation

| form | meaning |
| --- | --- |
| `x = y` | `x` is defined as `y` |
| `x \| y` | one of the alternatives |
| `[ x ]` | optional — `x` or nothing |
| `{ x }` | repetition — `x` zero or more times |
| `( x \| y )` | grouping |
| `x … y` | the range from `x` to `y` |
| `"..."` | the characters themselves — a terminal |

A production may be stated twice with `|` omitted; the alternatives accumulate.
Terminals are always quoted.

## Source text

A source file is UTF-8. A carriage return before a newline is read as part of
the newline, and a byte order mark at the very start of a file is skipped;
otherwise the compiler sees the bytes as they are. Identifiers are ASCII in
v0 — the Unicode question is an open item below.

Comments and blanks are equivalent — a comment is whitespace:

```ebnf
whitespace = blank | comment ;
blank      = " " | "\t" | newline ;
newline    = "\n" ;
comment    = "//" { any character except newline } [ newline ]
           | "/*" { comment | any character } "*/" ;
```

A `/*` comment runs to the matching `*/`, and a `/*` inside it opens another
one, which must also close — the production above is the intent; the two
characters of `*/` may not be split by anything, and `*` alone is content, so
`/* a * b */` is a whole comment. Nesting is deliberate: commenting out a
block that already contains a comment works. An unterminated comment is a
lexical error, not a run to end of file.

## Identifiers and keywords

```ebnf
identifier = letter { letter | digit | "_" } ;
letter     = "A" … "Z" | "a" … "z" ;
digit      = "0" … "9" ;
```

The keywords are:

| declarations | values | control flow |
| --- | --- | --- |
| `fn` `struct` `enum` `union` `trait` `impl` `type` `use` `pub` | `let` `const` `static` `mut` `dyn` `true` `false` | `if` `match` `for` `in` `break` `continue` `return` |

Two more are reserved, unused: `macro` and `defer`. Each names a feature the
design has considered and not carried — a user-defined macro system
(`14-macros.md`) and scoped cleanup — so the words wait for the decision
instead of being spent.

`true` and `false` are keywords because they are literals: `42` is lexed, not
resolved, and a boolean is the same kind of thing. The words that look like
keywords but are not:

- `self` — an ordinary parameter, by name convention (`05-traits.md`)
- `Self`, `main`, `Some`, `None` — names the language or the standard library
  defines, reached through the namespace rules like any other (`11-namespaces.md`,
  `12-projects.md`)
- attribute names — `packed`, `align`, `build`, `test`, and every user
  attribute are ordinary identifiers inside `#[...]` (`01-types.md`)

## Integer literals

```ebnf
integer   = decimal | hexadecimal | octal | binary ;

decimal     = digit { [ "_" ] digit } ;
hexadecimal = "0x" hex_digit { [ "_" ] hex_digit } ;
octal       = "0o" octal_digit { [ "_" ] octal_digit } ;
binary      = "0b" binary_digit { [ "_" ] binary_digit } ;

hex_digit    = digit | "a" … "f" | "A" … "F" ;
octal_digit  = "0" … "7" ;
binary_digit = "0" | "1" ;
```

An underscore separates digits and may not do anything else: it needs a digit
on each side, so `1_000_000` is one million, `1__0` and `1000_` are errors,
and `_1000` is an identifier. There is no suffix — `42u8` is not a form; the
type comes from context, an unsuffixed literal is `i32` (`01-types.md`), and
an explicit conversion is `@cast`. A sign is not part of the literal: `-1` is
the operator applied to `1`.

## Float literals

```ebnf
float     = digits "." digits [ exponent ]
          | digits exponent ;

digits    = digit { [ "_" ] digit } ;
exponent  = ( "e" | "E" ) [ "+" | "-" ] digits ;
```

A float needs digits on both sides of the dot — `1.` and `.5` are not forms;
write `1.0` and `0.5`. Only decimals carry floats; there is no hexadecimal
float. An unsuffixed float literal is `f32` (`01-types.md`).

## Byte and string literals

There is no `char` type (`01-types.md`): a single-quoted literal is a `u8`,
and it holds one byte.

```ebnf
byte_literal = "'" ( byte_character | escape ) "'" ;

plain_string = '"' { string_character | escape } '"' ;
c_string     = "c" '"' { string_character | escape } '"' ;
raw_string   = "r" '"' { raw_character } '"' ;
raw_c_string = "cr" '"' { raw_character } '"' ;

string_literal = plain_string | c_string | raw_string | raw_c_string ;

escape          = "\" ( "n" | "t" | "r" | "0" | "'" | '"' | "\" | "x" hex_digit hex_digit ) ;

byte_character   = any character except "'" | "\" | newline ;
string_character = any character except '"' | "\" | newline ;
raw_character    = any character except '"' | newline ;
```

The four string forms differ only in prefix and character set. `r` turns
every `\` into a plain byte and admits no escapes; `c` appends a `'\0'`
terminator — the meanings are `01-types.md`'s, and the prefixes combine as
`cr`. A prefix is part of the literal only when it touches it: an identifier
`c`, `r`, or `cr` immediately followed by `"` is the prefixed literal, and
that is the whole rule — an identifier by any other spelling, or with a
space before the quote, lexes as itself. No literal crosses a newline; a
string with a newline in it is written with `\n`.

The escapes are the whole set: `\n` `\t` `\r` `\0` `\'` `\"` `\\` and
`\xNN`, which takes any value through `0xFF` — the elements are bytes
(`01-types.md`). There is no `\u{...}`: a string literal keeps the bytes of
the source, and source is UTF-8, so a character beyond ASCII is written as
itself.

## Punctuation

`keyword` and `identifier` below are the sets defined above; every other
token is fixed here:

```ebnf
token = identifier | keyword | integer | float
      | byte_literal | string_literal
      | "(" | ")" | "[" | "]" | "{" | "}"
      | "," | ";" | ":" | "::" | "." | ".." | "..."
      | "->" | "?" | "@" | "$$" | "^^" | "#["
      | "+" | "-" | "*" | "/" | "%"
      | "^" | "&" | "|" | "!"
      | "<" | ">" | "<=" | ">=" | "==" | "!=" | "&&" | "||"
      | "=" | "+=" | "-=" | "*=" | "/=" ;
```

Lexing takes the longest match: `::` is one token, never two `:`, and
`a..b` reads `a`, `..`, `b`. This is what makes `^^` and `$$` single tokens
(`08-reflection.md`) and `#[` the only place `#` appears — a `#` not followed
by `[` is a lexical error. What the tokens mean in which position — the seven
uses of `[`, the three of `?` — is the expression pass.

## Open items

- Identifiers beyond ASCII — which Unicode set, and whether v0 wants it at
  all.
- Multiline string literals. The need is real (embedded text); the shape —
  a triple quote, an indent-aware form — is not yet chosen.
