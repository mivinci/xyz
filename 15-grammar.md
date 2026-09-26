# Grammar

This chapter states the language's grammar in EBNF. It arrives in three
passes: the first fixed the lexical grammar — the tokens. This pass fixes
expressions and types: the precedence levels, the three jobs of `?`, the
brackets. The third pass, declarations and statements, closes the language.

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

One consequence worth naming: there is no `>>` token, and none is wanted.
`is_same<i32, i32>>()` reads `>` `>` — nested generic arguments close with two
separate tokens (`05-traits.md`), which is only possible because the language
has no shift operators. If shifts are ever introduced, the lexer must split
`>>` and the grammar pays for it; that is an open item below, not a surprise
waiting in the token table.

## Expressions

The precedence levels, tightest first:

| level | operators | associativity |
| --- | --- | --- |
| postfix | `.f` `.0` `[...]` `(...)` `?` | chains left to right |
| unary | `*` `&` `&mut` `!` `-` `^^` `$$` `@name` | prefix |
| multiplicative | `*` `/` `%` | left |
| additive | `+` `-` | left |
| bit and | `&` | left |
| bit xor | `^` | left |
| bit or | `\|` | left |
| relational | `<` `>` `<=` `>=` `==` `!=` | **no chaining** |
| logical and | `&&` | left, short-circuit |
| logical or | `\|\|` | left, short-circuit |

Assignment is not on the table: it is a statement, not an expression
(`a = b = c` is not a form). The relational level does not chain — `a < b < c`
is a syntax error, and comparing two booleans is written with parentheses,
`(a < b) == (c < d)`; `&&` and `||` chain freely because a chain of them is
the common shape. `^^` and `$$` are prefix-only: `a ^^ b` is a syntax error
(`08-reflection.md`).

```ebnf
expression  = if_expr | match_expr | or_expr ;

if_expr     = [ "const" ] "if" expression block
              [ "else" ( if_expr | block ) ] ;

match_expr  = "match" expression "{" { arm "," } "}" ;
arm         = pattern "=>" ( block | expression ) ;

block       = "{" { statement } [ expression ] "}" ;
statement   = let_statement | assignment | expression ";" ;

or_expr     = and_expr { "||" and_expr } ;
and_expr    = cmp_expr { "&&" cmp_expr } ;
cmp_expr    = bitor_expr [ cmp_operator bitor_expr ] ;
bitor_expr  = bitxor_expr { "|" bitxor_expr } ;
bitxor_expr = bitand_expr { "^" bitand_expr } ;
bitand_expr = add_expr { "&" add_expr } ;
add_expr    = mul_expr { ( "+" | "-" ) mul_expr } ;
mul_expr    = unary_expr { ( "*" | "/" | "%" ) unary_expr } ;

unary_expr  = ( "*" | "&" [ "mut" ] | "!" | "-" | "^^" | "$$" ) unary_expr
            | postfix_expr ;

postfix_expr = primary_expr { postfix } ;
postfix      = "." ( identifier | integer )
             | "[" expression "]"
             | "[" [ expression ] ".." [ expression ] "]"
             | "(" [ arguments ] ")"
             | "?" ;

primary_expr = literal
            | path
            | "(" ")"
            | "(" expression { "," expression } [ "," ] ")"
            | array_literal
            | struct_literal
            | closure
            | builtin_call ;

path         = [ "::" ] segment { "::" segment } ;
segment      = identifier [ generic_args ] ;
generic_args = "<" generic_arg { "," generic_arg } [ "," ] ">" ;
generic_arg  = ( "$$" | "^^" ) postfix_expr | type ;

arguments    = argument { "," argument } [ "," ] ;
argument     = [ "..." ] expression ;

array_literal = "[" [ integer ] "]" [ "mut" ] type
                "{" [ expression { "," expression } [ "," ] ] "}" ;
struct_literal = path "{" [ field_init { "," field_init } [ "," ] ] "}" ;
field_init    = identifier ":" expression ;

closure     = "fn" "[" [ captures ] "]"
              "(" [ parameters ] ")" [ "->" type ] block ;
captures    = capture { "," capture } [ "," ] ;

builtin_call = "@" identifier ( "(" [ arguments ] ")"
               | generic_args "(" [ arguments ] ")" ) ;
```

An `if` is an expression: both branches have one type, and an `if` without an
`else` yields `()` on the untaken path, so it is written as a statement
(`10-iteration.md`). A `match` is an expression the same way, and an arm with
several statements is a block whose last expression is its value
(`09-match.md`). A block appears where a value is expected only as an `if`
branch, a `match` arm, or a function body — a bare `{ ... }` is not an
expression, so no binding takes one as its value. The condition of an `if`
and the scrutinee of a `match` are parsed where a block is about to open, so
a struct literal there must be parenthesized — `if Point{ x: 1, y: 2 } == q`
is not a form; `if (Point{ x: 1, y: 2 }) == q` is.

A `<` after an identifier in primary position starts generic arguments only
when a matching `>` and what generic arguments lead to — a `(`, a `::`, a
`.` — follow: `f<T>(x)` is a path with arguments called, `a < b` is a
comparison, and the parser decides by looking ahead to the closing `>`.
`is_same<i32, i32>::value` is the same rule one segment deeper — a path
segment may carry arguments anywhere along the path, and so may splice:
`is_same<$$t, u32>::value` passes a `type` value as an argument
(`08-reflection.md`). The `.` of a tuple index takes an integer — `a.0`,
never `a[0]` on a tuple (`01-types.md`).

The `...` of an argument is spread: `sum(...ts)` passes the elements of a
tuple one argument each (`04-generics.md`).

`let_statement`, `assignment`, `pattern`, and `parameters` appear above as
references into the third pass — declarations and statements, which closes
the grammar. A block may hold statements followed by one expression; what a
statement may be is settled there, and nothing here depends on the details.

## Types

Types have a grammar of their own, beside the expressions:

```ebnf
type          = result_type ;

result_type   = prefix_type [ "?" prefix_type ] ;
prefix_type   = "?" prefix_type
              | "*" [ "mut" ] prefix_type
              | "[" [ integer ] "]" [ "mut" ] prefix_type
              | primary_type ;

primary_type  = path
              | "(" ")"
              | "(" type { "," type } [ "," ] ")"
              | "fn" "(" [ parameters ] ")" [ "->" type ]
              | "dyn" path
              | "type" ;
```

`?T` is `Option<T>` and `E?T` is `Result<T, E>` — the `?` is a prefix when
the error side is empty and an infix when it is named (`05-traits.md`). The
grammar reads it in one step: a type is `E ? T` where `E` may be omitted,
and both sides nest — `E??T` is `Result<Option<T>, E>`, and an error type
that is itself optional parses, whether or not it makes sense.

`*T`, `*mut T`, `[N]T`, `[N]mut T`, `[]T`, `[]mut T` are all prefixes of the
type they wrap. Generic arguments are a suffix of a path — `Vec<u32>` — and
close with `>` tokens, one at a time, which is where the missing `>>` token
earns its keep.

### The `?` three ways

The one character with three jobs never shares a position: in a type it is
Option or Result, in an expression it is postfix propagation — `f()?` — and
nowhere else. A `?` at the head of an expression is a syntax error, and a
type never appears inside an expression without a marker (`@cast<u32>(x)` is
a builtin call; the type lives in its angle brackets). No lexer hint is
needed: the two grammars are disjoint, and position decides.

### The brackets

`[` has one use per position, and every one has its own production: a type
prefix (`[N]T`, `[]T`, `[]mut T`), an array literal at the head of an
expression (`[3]u32{1, 2, 3}` — the only `[` an expression may start with),
a postfix index or range (`a[i]`, `a[1..2]`, `a[..]`), the capture list
after `fn` (`fn[mut n]` — `fn` is a keyword, so the `[` is not ambiguous),
and `#[`, which the lexer has already taken. The capture list keeps its
brackets: no `fn|x|` — the `fn` prefix disambiguates on its own.

## Open items

- Shift and bitwise-not operators. The language has `&`, `^`, `|` and no
  `<<`, `>>`, `~` — no example ever wanted them, and their absence is what
  lets nested generics close with plain `>` tokens. Introducing shifts means
  splitting `>>` in the lexer (the Rust bill); if they arrive, they slot
  between additive and bit and, or wherever C puts them, and this chapter
  gains a row.
- Identifiers beyond ASCII — which Unicode set, and whether v0 wants it at
  all.
- Multiline string literals. The need is real (embedded text); the shape —
  a triple quote, an indent-aware form — is not yet chosen.
