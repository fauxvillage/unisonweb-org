---
title: Unison language reference
description: An informal reference for the Unison language

---

# Unison language reference

## Introduction

This document is an informal reference for the Unison language meant as an aid for Unison programmers as well as authors of implementations of the language. This isn't meant to be a tutorial or introductory guide to the language; it's more like a dry and unexciting tome you consult when you have questions about some aspect of the language. 🧐

This language reference, like the language it describes, is a work in progress and will be improved over time ([GitHub link](https://github.com/unisonweb/unisonweb-org/blob/master/src/data/docs/language-reference/README.md)). Contributions and corrections are welcome!

### A note on syntax

Unison is a language in which _programs are not text_. That is, the source of truth for a program is not its textual representation as source code, but its structured representation as an abstract syntax tree.

This document describes Unison in terms of its default (and currently, only) textual rendering into source code.

## Top-level declarations

A top-level declaration can appear at the _top level_ or outermost scope of a Unison file. It can be one of the following forms:

* A [term declaration][term], like `x = 42`.
* A [type declaration][type], like `type Optional a = None | Some a`.
* A [use clause][use], like `use .base` or `use math sqrt`.

[term]: #term-declarations
[type]: #user-defined-data-types
[use]:  #use-clauses

## Term declarations

A Unison term declaration (or "term binding") consists of an optional [type signature](#type-signatures), and a [term definition](#term-definition). For example:

```unison
timesTwo : Nat -> Nat
timesTwo x = x * 2
```

The first line in the above is a type signature. The type signature `timesTwo : Nat -> Nat` declares that the term named `timesTwo` is a function accepting an argument of type `Nat` and computes a value of type `Nat`. The type `Nat` is the type of 64-bit natural numbers starting from zero. See [Unison types](#types) for details.

The second line is the term definition. The `=` sign splits the definition into a _left-hand side_, which is the term being defined, and the _right-hand side_, which is the definition of the term.

The general form of a term binding is:

```unison
name : Type
name p_1 p_2 … p_n = expression
```

### Type signatures

`name : Type` is a type signature, where `name` is the name of the term being defined and `Type` is a [type](#types) for that term. The `name` given in the type signature and the `name` given in the definition must be the same.

Type signatures are optional. In the absence of a type signature, Unison will automatically infer the type of a term declaration. If a type signature is present, Unison will verify that the term has the type given in the signature.

### Term definition

A term definition has the form `f p_1 p_2 … p_n = e` where `f` is the name of the term being defined. The parameters `p_1` through `p_n` are the names of parameters, if any (if the term is a function), separated by spaces. The right-hand side of the `=` sign is any [Unison expression](#expressions).

The names of the parameters as well as the name of the term are bound as local variables in the expression on the right-hand side (also known as the _body_ of the function). When the function is called, the parameter names are bound to any arguments passed in the call. See [function application](#function-application) for details on the call semantics of functions.

If the term takes no arguments, the term has the value of the fully evaluated expression on the right-hand side and is not a function.

The expression comprising the right-hand side can refer to the name given to the definition in the left-hand side. In that case, it’s a recursive definition. For example:

```unison
sumUpTo : Nat -> Nat
sumUpTo n =
  if n < 2 then n
  else n + sumUpTo (drop n 1)
```

The above defines a function `sumUpTo` that recursively sums all the natural numbers less than some number `n`. As an example, `sumUpTo 3` is `1 + 2 + 3`, which is `6`.

Note: The expression `drop n 1` on line 4 above subtracts one from the natural number `n`. Since the natural numbers are not closed under subtraction (`n - 1` is an `Int`), we use the operation `drop` which has the convention that `drop 0 n = 0` for all natural numbers `n`. Unison's type system saves us from having to deal with negative numbers here.

### Operator definitions

[Operator identifiers](#identifiers) are valid names for Unison definitions, but the syntax for defining them is slightly different. For example, we could define a binary operator `**`:

``` unison
(**) x y = Float.pow x y
```

Or we could define it using infix notation:

``` unison
x ** y = Float.pow x y
```

If we want to give the operator a qualified name, we put the qualifier inside the parentheses:

``` unison
(MyNamespace.**) x y = Float.pow x y
```

Or if defining it infix:

``` unison
x MyNamespace.** y = Float.pow x y
```

The operator can be applied using either notation, no matter which way it's defined. See [function application](#function-application) for details.

## User-defined data types

A user-defined data type is introduced with the `type` keyword. (See [Types](#types) for an informal description of Unison's type system.)

For example:

``` unison
type Optional a = None | Some a
```

The `=` sign splits the definition into a _left-hand side_ and a _right-hand side_, much like term definitions.

The left-hand side is the data type being defined. It gives a name for the data type and declares a new [_type constructor_](#type-constructors) with that name (here it’s named `Optional`), followed by names for any type arguments (here there is one and it’s called `a`). These names are bound as type variables in the right-hand side. The right-hand side may also refer to the name given to the type in the left-hand side, in which case it is a recursive type declaration. Note that the fully saturated type construction `Optional Nat` is a type, whereas `Optional` by itself is a type constructor, not a type (it requires a type argument in order to construct a type).

The right-hand side consists of zero or more data constructors separated by `|`. These are _data constructors_ for the type, or ways in which values of the type can be constructed. Each case declares a name for a data constructor (here the data constructors are `None` and `Some`), followed by the **types** of the arguments to the constructor.

When Unison compiles a type definition, it generates a term for each data constructor. Here they are the terms `Optional.Some : a -> Optional a`, and `Optional.None : Optional a`. It also generates _patterns_ for matching on data (see [Pattern Matching](#match-expressions-and-pattern-matching)).

Note that these terms and patterns receive qualified names: if the type named `x.y.Z` has a data constructor `C`, the generated term and pattern for `C` will be named `x.y.Z.C`.

The general form of a type declaration is as follows:

``` unison
<unique<[<regular-identifier>]?>?> type TypeConstructor p1 p2 … pn
  = DataConstructor_1
  | DataConstructor_2
  ..
  | DataConstructor_n
```

The optional `unique` keyword introduces a [unique type](#unique-types), explained in the next section.

### Unique types

A type declaration gives a name to a type, but Unison does not uniquely identify a type by its name. Rather, the [hash](#hashes) of a type's definition identifies the type. The hash is based on the _structure_ of the type definition, with all identifiers removed.

For example, Unison considers these type declarations to declare _the exact same type_, even though they give different names to both the type constructor and the data constructors:

``` unison
type Optional a = Some a | None

type Maybe a = Just a | Nothing
```

So a value `Some 10` and a value `Just 10` are in fact the same value and these two expressions have the same type. Even though one nominally has the type `Optional Nat` and the other `Maybe Nat`, Unison understands that as the type `#5isltsdct9fhcrvu ##Nat`.

This is not always what you want. Sometimes you want to give meaning to a type that is more than just its structure. For example, it might be confusing that these two types are identical:

``` unison
type Suit = Hearts | Spades | Diamonds | Clubs

type Direction = North | South | East | West
```

Unison will consider every unary type constructor with four nullary data constructors as identical to these declarations. So Unison will not stop us providing a `Direction` where a `Suit` is expected.

The `unique` keyword solves this problem:

``` unison
unique type Suit = Hearts | Spades | Diamonds | Clubs

unique type Direction = North | South | East | West
```

When compiling these declarations, Unison will generate a [universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier) for the type and use that identifier when generating the hash for the type. As a result, the type gets a hash that is universally unique.

### Record types

In the type declarations discussed above, the arguments to each data constructor are nameless.  For example:

``` unison
type Point = Point Nat Nat
```

Here, the data type `Point` has a constructor `Point.Point`, with two arguments, both of type `Nat`.  The arguments have no name, so they are identified positionally, for example when creating a value of this type, like `Point.Point 1 2`.

Types with a single data constructor can also be defined in the following style, in which case they are called _record types_.

``` unison
type Point = { x : Nat, y : Nat }
```

This assigns names to each argument of the constructor.  The effect of this is to generate some accessor methods, to help get, set, and modify each field.

``` unison
Point.x        : Point -> Nat
Point.x.modify : (Nat -> Nat) -> Point -> Point
Point.x.set    : Nat -> Point -> Point
Point.y        : Point -> Nat
Point.y.modify : (Nat -> Nat) -> Point -> Point
Point.y.set    : Nat -> Point -> Point
```

> 👉 Note that `set` and `modify` are returning new, modified copies of the input record - there's no mutation of values in Unison.

There's currently no special syntax for creating or pattern matching on records.  That works the same as for regular data types:

``` unison
p = Point.Point 1 2
px = match p with
       Point.Point x _ -> x
```

### User-defined abilities

A user-defined _ability_ declaration has the following general form:

``` unison
ability A p_1 p_2 … p_n where
  Request_1 : Type_1
  Request_2 : Type_2
  Request_n : Type_n

```

This declares an _ability type constructor_ `A` with type parameters `p_1` through `p_n`, and _request constructors_ `Request_1` through `Request_n`.

See [Abilities and Ability Handlers](#abilities) for more on user-defined abilities.

## Expressions

This section describes the syntax and informal semantics of Unison expressions.

Unison's evaluation strategy for expressions is [Applicative Order Call-by-Value](https://en.wikipedia.org/wiki/Evaluation_strategy#Applicative_order). See [Function application](#function-application) for details.

### Basic lexical forms

See the sections on:

* [Identifiers](#identifiers), for example `foo`, `foo.bar`, and `+`.
* [Blocks and statements](#blocks), for example: `x = 42`.
* [Literals](#literals), for example: `1`, `"hello"`, `[1,2,3]`.
* [Comments](#comments), for example `-- this is a comment`.
* [Pattern matching](#match-expressions-and-pattern-matching), for example `match (x,y) with (1, "hi") -> 42`.

### Identifiers

Unison identifiers come in two flavors:

1. _Regular identifiers_ start with an alphabetic unicode character, emoji (which is any unicode character between 1F400 and 1FAFF inclusive), or underscore (`_`), followed by any number of alphanumeric characters, emoji, or the characters `_`, `!`, or `'`. For example, `foo`, `_bar4`, `qux'`, and `set!` are valid regular identifiers.
2. _Operators_ consist entirely of the characters `!$%^&*-=+<>.~\\/|:`. For example, `+`, `*`, `<>`, and `>>=` are valid operators.

#### Namespace-qualified identifiers

The above describes _unqualified_ identifiers. An identifier can also be _qualified_. A qualified identifier consists of a _qualifier_ or _namespace_, followed by a `.`, followed by either a regular identifier or an operator. The qualifier is one or more regular identifiers separated by `.`. For example `Foo.Bar.baz` is a qualified identifier where `Foo.Bar` is the qualifier.

#### Absolutely qualified identifiers

Namespace-qualified identifiers described above are relative to a “current” namespace, which the programmer can set (and defaults to the root of the global namespace). To ignore the current namespace, an identifier can have an _absolute qualifier_. An absolutely qualified name begins with a `.`. For example, the name `.base.List` always refers to the name `.base.List`, regardless of the current namespace, whereas the name `base.List` will refer to `foo.base.List` if the current namespace is `foo`.

Note that [operator identifiers](#identifiers) may contain the character `.`. In order for this to not create ambiguity, the rule is as follows:

1. `.` by itself is always an operator.
2. Any other identifier beginning with `.` is an absolutely qualified identifier.
3. A `.` immediately following a namespace is always a namespace separator.
4. Otherwise a `.` is treated as part of an operator identifier.

if `.` is followed by whitespace or another operator character, the `.` is treated like an operator character. If it's followed by a [regular identifier](#identifiers) character, it's treated as a namespace separator.

#### Hash-qualified identifiers

Any identifier, including a namespace-qualified one, can appear _hash-qualified_. A hash-qualified identifier has the form `x#h` where `x` is an identifier and `#h` is a [hash literal](#hashes). The hash disambiguates names that may refer to more than one thing.

#### Reserved words

The following names are reserved by Unison and cannot be used as identifiers: `=`, `:`, `->`, `'`, `|`, `!`, `'`, `if`, `then`, `else`, `forall`, `handle`, `unique`, `where`, `use`, `&&`, `||`, `true`, `false`, `type`, `ability`, `alias`, `let`, `namespace`, `cases`, `match`, `with`, `termLink`, `typeLink`.

### Name resolution and the environment

During typechecking, Unison substitutes free variables in an expression by looking them up in an environment populated from a _codebase_ of available definitions. A Unison codebase is a database of term and type definitions, indexed by [hashes](#hashes) and names.

A name in the environment can refer to either terms or types, or both (a type name can never be confused with a term name).

#### Suffix-based name resolution

If the list of _segments_ of a name (`base.List.map` has the segments `[base,List,map]`)  is a suffix of exactly one fully qualified name in the environment, Unison substitutes that name in the expression with a reference to the definition. For example, the fully qualified name `.base.List.map` could be referenced via `base.List.map`, `List.map` as long as no other definitions end in `base.List.map` or `List.map`. This reduces the number of [imports](#use) needed and cuts down on needing to remember the fully qualified names for definitions. ("Was it `base.List.map` or `util.List.map`?")

[Hash literals](#hashes) in the program are substituted with references to the definitions in the environment whose hashes they match.

If a free term variable in the program cannot be found in the environment and is not the name of another term in scope in the program itself, or if an free variable matches more than one name (it’s ambiguous), Unison tries _type-directed name resolution_.

#### Type-directed name resolution

During typechecking, if Unison encounters a free term variable that is not a term name in the environment, Unison attempts _type-directed name resolution_, which:

1. Finds term definitions in the environment whose _unqualified_ name is the same as the free variable.
2. If exactly one of those terms has a type that conforms to the expected type of the variable (the type system has always inferred this type already at this point), perform that substitution and resume typechecking.

If name resolution is unable to find the definition of a name, or is unable to disambiguate an ambiguous name, Unison reports an error.<a id="blocks"></a>

### Blocks and statements

A block is an expression that has the general form:

``` unison
statement_1
statement_2
...
statement_n
expression

```

A block can have zero or more statements, and the value of the whole block is the value of the final `expression`. A statement is either:

1. A [term definition](#term-definition) which defines a term within the scope of the block. The definition is not visible outside this scope, and is bound to a local name. Unlike top-level definitions, a block-level definition does not result in a hash, and cannot be referenced with a [hash literal](#hashes).
2. A [Unison expression](#expressions). In particular, blocks often contain _action expressions_, which are expressions evaluated solely for their effects. An action expression has type `{A} T` for some ability `A` (see [Abilities and Ability Handlers](#abilities)) and some type `T`.
3. A [`use` clause](#use).

An example of a block (this evaluates to `16`):

``` unison
x = 4
y = x + 2
f a = a + y
f 10

```

A number of language constructs introduce blocks. These are detailed in the relevant sections of this reference. Wherever Unison expects an expression, a block can be introduced with the  `let` keyword:

``` unison
let <block>

```

Where `<block>` denotes a block as described above.

#### The lexical syntax of blocks

The standard syntax expects statements to appear in a line-oriented layout, where whitespace is significant.

The opening keyword (`let`, `if`, `then`, or `else`, for example) introduces the block, and the position of the first character of the first statement in the block determines the top-left corner of the block. The beginning of each statement in the block must be lined up exactly with the left edge of the block. The first non-whitespace character that appears to the left of that edge (i.e. outdented) ends the block. Certain keywords also end blocks. For example, `then` ends the block introduced by `if`.

A statement or expression in a block can continue for more than one line as long as each line of the statement is indented further than the first character of the statement or expression.

For example, these are valid indentations for a block:

``` unison
let
  x = 1
  y = 2
  x + y


let x = 1
    y = 2
    x + y

```

Whereas these are incorrect:

``` unison
let x = 1
  y = 2
  x + y

let x = 1
     y = 2
       x + y

```

##### Syntactic precedence

Keywords that introduce blocks bind more tightly than [function application](#function-application). So `f let x` is the same as `f (let x)` and `f if b then p else q` is the same as `f (if b then p else q)`.

Block keywords bind more tightly than [delayed computations](#delayed-computations) syntax. So `'let x` is the same as `_ -> let x` and `!if b then p else q` is the same as `(if b then p else q) ()`.

Blocks eagerly consume expressions, so `if b then p else q + r` is the same as `if b then p else (q + r)`.

### Literals

A literal expression is a basic form of Unison expression. Unison has the following types of literals:

* A _64-bit unsigned integer_ of type `.base.Nat` (which stands for _natural number_) consists of digits from 0 to 9. The smallest `Nat` is `0` and the largest is `18446744073709551615`.
* A _64-bit signed integer_ of type `.base.Int` consists of a natural number immediately preceded by either `+` or `-`. For example, `4` is a `Nat`, whereas `+4` is an `Int`. The smallest `Int` is `-9223372036854775808` and the largest is `+9223372036854775807`.
* A _64-bit floating point number_ of type `.base.Float` consists of an optional sign (`+`/`-`), followed by two natural numbers separated by `.`. Floating point literals in Unison are [IEEE 754-1985](https://en.wikipedia.org/wiki/IEEE_754-1985) double-precision numbers. For example `1.6777216` is a valid floating point literal.
* A _text literal_ of type `.base.Text` is any sequence of Unicode characters between pairs of `"`. The escape character is `\`, so a `"` can be included in a text literal with the escape sequence `\"`. The full list of escape sequences is given in the [Escape Sequences](#escape-sequences) section below. For example, `"Hello, World!"` is a text literal. A text literal can span multiple lines. Newlines do not terminate text literals, but become part of the literal text.
* A _character literal_ of type `.base.Char` consists of a `?` character marker followed by a single Unicode character, or a single [escape sequence](#escape-sequences). For example, `?a`, `?🔥` or `?\t`.
* There are two _Boolean literals_: `true` and `false`, and they have type `Boolean`.
* A _hash literal_ begins with the character `#`. See the section **Hashes** for details on the lexical form of hash literals. A hash literal is a reference to a term or type. The type or term that it references must have a definition whose hash digest matches the hash in the literal. The type of a hash literal is the same as the type of its referent. `#a0v829` is an example of a hash literal.
* A _literal list_ has the general form `[v1, v2, ... vn]` where `v1` through `vn` are expressions. A literal list may be empty. For example, `[]`, `[x]`, and `[1,2,3]` are list literals. The expressions that form the elements of the list all must have the same type. If that type is `T`, then the type of the list literal is `.base.List T` or `[T]`.
* A _function literal_ or _lambda_ has the form `p1 p2 ... pn -> e`, where `p1` through `pn` are [regular identifiers](#identifiers) and `e` is a Unison expression (the _body_ of the lambda). The variables `p1` through `pn` are local variables in `e`, and they are bound to any values passed as arguments to the function when it’s called (see the section [Function Application](#function-application) for details on call semantics). For example `x -> x + 2` is a function literal.
* A _tuple literal_ has the form `(v1,v2, ..., vn)` where `v1` through `vn` are expressions. A value `(a,b)` has type `(A,B)` if `a` has type `A` and `b` has type `B`. The expression `(a)` is the same as the expression `a`. The nullary tuple `()` (pronounced “unit”) is of the trivial type `()`. See [tuple types](#tuple-types) for details on these types and more ways of constructing tuples.

#### Documentation literals

Documentation blocks have type `Doc` (documentation is a first-class value in the language).`{{` starts a documentation block and `}}` finishes it. For example:

```unison
List.take.apiDocs : Doc
List.take.apiDocs = {{
  ``List.take n [1,2,3]`` returns the first `n` elements of
  a list. This is efficient and takes just `O(log n)` operations.

  ## Example

    {{ examples.List.take.ex1 }}

  ## Also see

    * The {type List} type and associated docs 
    * The {List.drop} function with signature @inlineSignature{List.drop}
    * More about finger trees (used to implement {type List}) here: {fingerTrees.doc}
}}

```

More specifically, some important Doc features are:

* Links to definitions are done with single open and close braces. `{List.drop}` is a term link, and `{type List}` is a type link.
* `@signature{List.take}` or `@inlineSignature{List.take}` expands to the type signature of List.take either as a block or inline, respectively.
* `@source{List.map}` expands to the full source of List.map
*  To insert another Doc value into another Doc, use nested double braces. `{{I am a doc {{thisDocValueWillBeDisplayed}} }}`
* `@eval{someDefinition}` expands to the result of evaluating `someDefinition`, which must be a pre-existing definition in the codebase (it can't be an arbitrary expression). You can evaluate multi-line codeblocks with the triple backtick syntax ```` ``` multipleLines ``` ```` 

#### Escape sequences

Text literals can include the following escape sequences:

* `\0` = null character
* `\a` = alert (bell)
* `\b` = backspace
* `\f` = form feed
* `\n` = new line
* `\r` = carriage return
* `\t` = horizontal tab
* `\v` = vertical tab
* `\s` = space
* `\\` = literal `\` character
* `\'` = literal `'` character
* `\"` = literal `"` character

### Comments

A line comment starts with `--` and is followed by any sequence of characters. A line that contains a comment can’t contain anything other than a comment and whitespace. Line comments are currently ignored by Unison.

Multi-line comments are supported by starting a comment block with `{-` and ending it with `-}`. 

A line starting with `---` and containing no other characters is a _fold_. Any text below the fold is ignored by Unison.

### Type annotations

A type annotation has the form `e:T` where `e` is an expression and `T` is a type. This tells Unison that `e` should be of type `T` (or a subtype of type `T`), and Unison will check whether this is true. It's a type error for the actual type of `e` to be anything other than a type that conforms to `T`.

### Parenthesized expressions

Any expression can appear in parentheses, and an expression `(e)` is the same as the expression `e`. Parentheses can be used to delimit where an expression begins and ends. For example `(f : P -> Q) y` is an application of the function `f` of type `P -> Q` to the argument `y`. The parentheses are needed to tell Unison that `y` is an argument to `f`, not a part of the type annotation expression.

### Function application

A function application `f a1 a2 an` applies the function `f` to the arguments `a1` through `an`.

The above syntax is valid where `f` is a [regular identifier](#identifiers). If the function name is an operator such as `*`, then the syntax for application is infix :  `a1 * a2`. Any operator can be used in prefix position by surrounding it in parentheses: `(*) a1 a2`. Any [regular identifier](#identifiers) can be used infix by surrounding it in backticks: ``a1 `f` a2``.

All Unison functions are of arity 1. That is, they take exactly one argument. An n-ary function is modeled either as a unary function that returns a further function (a partially applied function) which accepts the rest of the arguments, or as a unary function that accepts a tuple.

Function application associates to the left, so the expression `f a b` is the same as `(f a) b`. If `f` has type `T1 -> T2 -> Tn` then `f a` is well typed only if `a` has type `T1`. The type of `f a` is then `T2 -> Tn`. The type constructor of function types, `->`, associates to the right. So `T1 -> T2 -> Tn` parenthesizes as `T1 -> (T2 -> TN)`.

The evaluation semantics of function application is applicative order [Call-by-Value](https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value). In the expression `f x y`, `x` and `y` are fully evaluated in left-to-right order, then `f` is fully evaluated, then `x` and `y` are substituted into the body of `f`, and lastly the body is evaluated.

An exception to the evaluation semantics is [Boolean expressions](#boolean-expressions), which have non-strict semantics.

Unison supports [proper tail calls](https://en.wikipedia.org/wiki/Tail_call) so function calls in tail position do not grow the call stack.

#### Syntactic precedence of operators and prefix function application

All operators and infix function applications currently have the same precedence, and are parsed left-associative. Use parentheses to obtain a different grouping. So for instance:

* `1 + 3 * 4` is parsed `(1 + 3) * 4`. You can group it differently using parentheses: `1 + (3 * 4)`
* `a + b < c * d` is parsed as `((a + b) < c) * d`.  You can group it differently using parentheses: `a + b < (c * d)`
* ``x `Nat.drop` 1 + 11`` is parsed as ``(x `Nat.drop` 1) + 11``.

When your code is printed back to you by Unison, it will be displayed with minimal necessary parentheses, so if Unison's default syntax later supports operator precedence, old definitions written originally with more parentheses will get displayed with only the parentheses currently needed for the current precedence settings.

Prefix function application:

* Binds more tightly than infix operators. So `f x + g y` is the same as `(f x) + (g y)`.
* Binds less tightly than keywords that introduce [blocks](#blocks). So `f let x` is the same as `f (let x)` and `f if b then p else q` is the same as `f (if b then p else q)`
* Binds less tightly than `'` and `!` (see [delayed computations](#delayed-computations)), so `'f x y` is the same as `(_ -> f) x y` and `!f x y` is the same as `f () x y`.

### Boolean expressions

A Boolean expression has type `Boolean` which has two values, `true` and `false`.

#### Conditional expressions

A _conditional expression_ has the form `if c then t else f`, where `c` is an expression of type `Boolean`, and `t` and `f` are expressions of any type, but `t` and `f` must have the same type.

Evaluation of conditional expressions is non-strict. The evaluation semantics of `if c then t else f` are:

* The condition `c` is always evaluated.
* If `c` evaluates to `true`, the expression `t`  is evaluated and `f` remains unevaluated. The whole expression reduces to the value of `t`.
* If `c` evaluates to `false`, the expression `f` is evaluated and `t` remains unevaluated. The whole expression reduces to the value of `f`.

The keywords `if`, `then`, and `else` each introduce a [Block](#blocks-and-statements)  as follows:

``` unison
if
  <block>
then
  <block>
else
  <block>

```

#### Boolean conjunction and disjunction

A _Boolean conjunction expression_ is a `Boolean` expression of the form `a && b` where `a` and `b` are `Boolean` expressions. Note that `&&` is not a function, but built-in syntax.

The evaluation semantics of `a && b` are equivalent to `if a then b else false`.

A _Boolean disjunction expression_ is a `Boolean` expression of the form `a || b` where `a` and `b` are `Boolean` expressions. Note that `||` is not a function, but built-in syntax.

The evaluation semantics of `a || b` are equivalent to `if a then true else b`.

### Delayed computations

An expression can appear _delayed_ as `'e`, which is the same as `_ -> e`. If `e` has type `T`, then `'e` has type `forall a. a -> T`.

If `c` is a delayed computation, it can be _forced_ with `!c`, which is the same as `c ()`. The expression `c` must conform to a type `() -> t` for some type `t`, in which case `!c` has type `t`.

Delayed computations are important for writing expressions that require [abilities](#abilities). For example:

``` unison
use io

program : '{IO} ()
program = 'let
  printLine "What is your name?"
  name = !readLine
  printLine ("Hello, " ++ name)

```

This example defines a small I/O program. The type `{IO} ()` by itself is not allowed as the type of a top-level definition, since the `IO` ability must be provided by a handler, see [abilities and ability handlers](#abilities)). Instead, `program` has the type `'{IO} ()` (note the `'` indicating a delayed computation). Inside a handler for `IO`, this computation can be forced with `!program`.

Inside the program, `!readLine` has to be forced, as the type of `io.readLine` is `'{IO} Text`, a delayed computation which, when forced, reads a line from standard input.

#### Syntactic precedence

The reserved symbols `'` and `!` bind more tightly than function application, So `'f x` is the same as `(_ -> f) x` and `!x + y` is the same as `(x ()) + y`.

These symbols bind less tightly than keywords that introduce blocks, so `'let x` is the same as `_ -> let x` and `!if b then p else q` is the same as `(if b then p else q) ()`.

Additional `'` and `!` combine in the obvious way:

  * `''x` is the same as `(_ -> (_ -> x))` or `(_ _ -> x)`.
  * `!!x` is the same as `x () ()`.
  * `!'x` and `'!x` are both the same as `x`.

You can of course use parentheses to precisely control how `'` and `!` get applied.

### Match expressions and pattern matching

A _match expression_ has the general form:

``` unison
match e with
  pattern_1 -> block_1
  pattern_2 -> block_2
  ...
  pattern_n -> block_n
```

Where `e` is an expression, called the _scrutinee_ of the match expression, and each _case_ has a [pattern to match against the value of the scrutinee](#blank-patterns) and a [block](#blocks) to evaluate in case it matches.

The evaluation semantics of match expressions are as follows:

1. The scrutinee is evaluated.
2. The first pattern is evaluated and matched against the value of the scrutinee.
3. If the pattern matches, any variables in the pattern are substituted into the block to the right of its `->` (called the _match body_) and the block is evaluated. If the pattern doesn’t match then the next pattern is tried and so on.

It's possible for Unison to actually evaluate cases in a different order, but such evaluation should always have the same observable behavior as trying the patterns in sequence.

It is an error if none of the patterns match. In this version of Unison, the  error occurs at runtime. In a future version, this should be a compile-time error.

Unison provides syntactic sugar for match expressions in which the scrutinee is the sole argument of a lambda expression:

```unison
cases
  pattern_1 -> block_1
  pattern_2 -> block_1
  ...
  pattern_n -> block_n

-- equivalent to
e -> match e with
  pattern_1 -> block_1
  pattern_2 -> block_1
  ...
  pattern_n -> block_n
```

A _pattern_ has one of the following forms:

#### Blank patterns

A _blank pattern_ has the form `_`. It matches any expression without creating a variable binding.

For example:

``` unison
match 42 with
  _ -> "Always matches"
```

#### Literal patterns

A _literal pattern_ is a literal `Boolean`, `Nat`, `Int`, `Char`, or `Text`. A literal pattern matches if the scrutinee has that exact value.

For example:

``` unison
match 2 + 2 with
  4 -> "Matches"
  _ -> "Doesn't match"
```

#### Variable patterns

A _variable pattern_ is a [regular identifier](#identifiers) and matches any expression. The expression that it matches will be bound to that identifier as a variable in the match body.

For example, this expression evaluates to `3`:

``` unison
match 1 + 1 with
  x -> x + 1
```

#### As-patterns

An _as-pattern_ has the form `v@p` where `v` is a [regular identifier](#identifiers) and `p` is a pattern. This pattern matches if `p` matches, and the variable `v` will be bound in the body to the value matching `p`.

For example, this expression evaluates to `3`:

``` unison
match 1 + 1 with
  x@4 -> x * 2
  y@2 -> y + 1
  _   -> 22
```

#### Constructor patterns

A _constructor pattern_ has the form `C p1 p2 ... pn` where `C` is the name of a data constructor in scope, and `p1` through `pn` are patterns such that `n` is the arity of `C`. Note that `n` may be zero. This pattern matches if the scrutinee reduces to a fully applied invocation of the data constructor `C` and the patterns `p1` through `pn` match the arguments to the constructor.

For example, this expression uses `Some` and `None`, the constructors of the `Optional` type, to return the 3rd element of the list `xs` if present or `0` if there was no 3rd element.

``` unison
match List.at 3 xs with
  None -> 0
  Some x -> x
```

#### List patterns

A _list pattern_ matches a `List t` for some type `t` and has one of three forms:

1. `head +: tail` matches a list with at least one element. The pattern `head` is matched against the first element of the list and `tail` is matched against the suffix of the list with the first element removed.
2. `init :+ last` matches a list with at least one element. The pattern `init` is matched against the prefix of the list with the last element removed, and `last` is matched against the last element of the list.
3. A _literal list pattern_ has the form `[p1, p2, ... pn]` where `p1` through `pn` are patterns. The patterns `p1` through `pn` are  matched against the elements of the list. This pattern only matches if the length of the scrutinee is the same as the number of elements in the pattern. The pattern `[]` matches the empty list.
4. `part1 ++ part2` matches a list which composed of the concatenation of `part1` and `part2`. At least one of `part1` or `part2` must be a pattern with a known list length, otherwise it's unclear where the list is being split. For instance, `[x,y] ++ rest` is okay as is `start ++ [x,y]`, but just `a ++ b` is not allowed.

Examples:

``` unison
first : [a] -> Optional a
first as = match as with
  h +: _ -> Some h
  [] -> None

last : [a] -> Optional a
last as = match as with
  _ :+ l -> Some l
  [] -> None

exactlyOne : [a] -> Boolean
exactlyOne a = match a with
  [_] -> true
  _   -> false

lastTwo : [a] -> Optional (a,a)
lastTwo a = match a with
  start ++ [a,a2] -> Some (a,a2)
  _ -> None

firstTwo : [a] -> Optional (a,a)
firstTwo a = match a with
  [a,a2] ++ rest -> Some (a,a2)
  _ -> None
```

#### Tuple patterns

 A _tuple pattern_ has the form `(p1, p2, ... pn)` where `p1` through `pn` are patterns. The pattern matches if the scrutinee is a tuple of the same arity as the pattern and `p1` through `pn` match against the elements of the tuple. The pattern `(p)` is the same as the pattern `p`, and the pattern `()` matches the literal value `()` of the trivial type `()` (both pronounced “unit”).

For example, this expression evaluates to `4`:

``` unison
match (1,2,3) with
  (a,_,c) -> a + c
```

#### Ability patterns (or `Request` patterns)

An _ability pattern_ only appears in an _ability handler_ and has one of two forms (see [Abilities and ability handlers](#abilities) for details):

1. `{C p1 p2 ... pn -> k}` where `C` is the name of an ability constructor in scope, and `p1` through `pn` are patterns such that `n` is the arity of `C`. Note that `n` may be zero. This pattern matches if the scrutinee reduces to a fully applied invocation of the ability constructor `C` and the patterns `p1` through `pn` match the arguments to the constructor.  The scrutinee must be of type `Request A T` for some ability `{A}` and type `T`. The variable `k` will be bound to the continuation of the program. If the scrutinee has type `Request A T` and `C` has type `X ->{A} Y`, then `k` has type `Y -> {A} T`.
2. `{p}` where `p` is a pattern. This matches the case where the computation is _pure_ (the value of type `Request A T` calls none of the constructors of the ability `{A}`). A pattern match on an `Request` is not complete unless this case is handled.

See the section on [abilities and ability handlers](#abilities) for examples of ability patterns.

#### Guard patterns

A _guard pattern_ has the form `p | g` where `p` is a pattern and `g` is a Boolean expression that may reference any variables bound in `p`. The pattern matches if `p` matches and `g` evaluates to `true`.

For example, the following expression evaluates to 6:

``` unison
match 1 + 2 with
  x | x == 4 -> 0
  x | x + 1 == 4 -> 6
  _ -> 42
```

## Hashes

A _hash_ in Unison is a 512-bit SHA3 digest of a term or a type's internal structure, excluding all names. The textual representation of a hash is its [base32Hex](https://github.com/multiformats/multibase#multibase-table-v100-rc-semver) Unicode encoding.

Unison attributes a hash to every term and type declaration, and the hash may be used to unambiguously refer to that term or type in all contexts. As far as Unison is concerned, the hash of a term or type is its _true name_.

### Literal hash references

A term, type, data constructor, or ability constructor may be unambiguously referenced by hash. Literal hash references have the following structure:

* A _term definition_ has a hash of the form `#x` where `x` is the base32Hex encoding of the hash of the term. For example `#a0v829`.
* A term or type definition that’s part of a _cycle of mutually recursive definitions_ hashes to the form `#x.n` where `x` is the hash of the cycle and `n` is the term or type’s index in its cycle. A cycle has a canonical order determined by sorting all the members of the cycle by their individual hashes (with the cycle removed).
* A data constructor hashes to the form `#x#c` where `x` is the hash of the data type definition and `c` is the index of that data constructor in the type definition.
* A data constructor in a cyclic type definition hashes to the form `#x.n#c` where `#x.n` is the hash of the data type and `c` is the data constructor’s index in the type definition.
* A _built-in reference_ to a Unison built-in term or type `n` has a hash of the form `##n`. `##Nat` is an example of a built-in reference.

### Short hashes

A hash literal may use a prefix of the base32Hex encoded SHA3 digest instead of the whole thing. For example the programmer may use a short hash like `#r1mtr0` instead of the much longer 104-character representation of the full 512-bit hash. If the short hash is long enough to be unambiguous given the [environment](#name-resolution-and-the-environment), Unison will substitute the full hash at compile time. When rendering code as text, Unison may calculate the minimum disambiguating hash length before rendering a hash.

## Types

This section describes informally the structure of types in Unison. See also the section titled [User-defined types](#type-declarations) for detailed information on how to define new data types.

Formally, Unison’s type system is an implementation of the system described by Joshua Dunfield and Neelakantan R. Krishnaswami in their 2013 paper [Complete and Easy Bidirectional Typechecking for Higher-Rank Polymorphism](https://arxiv.org/abs/1306.6032).

Unison extends that type system with, [pattern matching](#match-expressions-and-pattern-matching), [scoped type variables](#scoped-type-variables), _ability types_ (also known as _algebraic effects_). See the section on [Abilities](#abilities) for details on ability types.

Unison attributes a type to every valid expression. For example:

* `4 < 5` has type `Boolean`
* `42 + 3` has type `Nat`,
* `"hello"` has type `Text`
* the list `[1,2,3]` has type `[Nat]`
* the function `(x -> x)` has type `forall a. a -> a`

The meanings of these types and more are explained in the sections below.

A full treatise on types is beyond the scope of this document. In short, types help enforce that Unison programs make logical sense. Every expression must be well typed, or Unison will give a compile-time type error. For example:

* `[1,2,3]` is well typed, since lists require all elements to be of the same type.
* `42 + "hello"` is not well typed, since the type of `+` disallows adding numbers and text together.
* `printLine "Hello, World!"` is well typed in some contexts and not others. It's a type error for instance to use I/O functions where an `IO` [ability](#abilities) is not provided.

Types are of the following general forms.

### Type variables

Type variables are [regular identifiers](#identifiers) beginning with a lowercase letter. For example `a`, `x0`, and `foo` are valid type variables.

### Polymorphic types

A _universally quantified_ or _polymorphic_ type has the form `forall v1 v2 vn. t`, where `t` is a type. The type `t` may involve the variables `v1` through `vn`.

The symbol `∀` is an alias for `forall`.

A type like `forall x. F x` can be written simply as `F x` (the `forall x` is implied) as long as `x` is free in `F x` (it is not bound by an outer scope; see [Scoped type variables](#scoped-type-variables) below).

A polymorphic type may be _instantiated_ at any given type. For example, the empty list `[]` has type `forall x. [x]`. So it's a type-polymorphic value. Its type can be instantiated at `Int`, for example, which binds `x` to `Int` resulting in `[Int]` which is also a valid type for the empty list. In fact, we can say that the empty list `[]` is a value of type `[x]` _for all_ choices of element type `e`, hence the type `forall x. [x]`.

Likewise the identity function `(x -> x)`, which simply returns its argument, has a polymorphic type `forall t. t -> t`. It has type `t -> t` for all choices of `t`.

### Scoped type variables

Type variables introduced by a type signature for a term remain in scope throughout the definition of that term.

For example in the following snippet, the type annotation `temp:x` is telling Unison that `temp` has the type `x` which is bound in the type signature, so `temp` and `a` have the same type.

``` unison
ex1 : x -> y -> x
ex1 a b =
  -- refers to the type x in the outer scope
  temp : x
  temp = a
  a
```

To explicitly shadow a type variable in scope, the variable can be reintroduced in the inner scope by a `forall` binder, as follows:

``` unison
ex2 : x -> y -> x
ex2 a b =
  -- doesn’t refer to x in outer scope
  id : ∀ x . x -> x
  id v = v
  temp = id 42
  id a
```

Note that here the type variable `x` in the type of `id` gets instantiated to two different types. First `id 42` instantiates it to `Nat`, then `id a`, instantiates it to the outer scope's type `x`.

### Type constructors

Just as values are built using data constructors, types are built from _type constructors_. Nullary type constructors like `Nat`, `Int`, `Float` are already types, but other type constructors like `List` and `->` (see [built-in type constructors](#built-in-type-constructors)) take type parameters in order to yield types. `List` is a unary type constructor, so it takes one type (the type of the list elements), and `->` is a binary type constructor. `List Nat` is a type and `Nat -> Int` is a type.

### Kinds of Types

Types are to values as _kinds_ are to type constructors. Unison attributes a kind to every type constructor, which is determined by its number of type parameters and the kinds of those type parameters.

A type must be well kinded, just like an expression must be well typed, and for the same reason. However, there is currently no syntax for kinds and they do not appear in Unison programs (this will certainly change in a future version of Unison).

Unison’s kinds have the following forms:

* A nullary type constructor or ordinary type has kind `Type`.
* A type constructor has kind `k1 -> k2` where `k1` and `k2` are kinds.

For example `List`, a unary type constructor, has kind `Type -> Type` as it takes a type and yields a type. A binary type constructor like `->` has kind `Type -> Type -> Type`, as it takes two types (it actually takes a type and yields a partially applied unary type constructor that takes the other type). A type constructor of kind `(Type -> Type) -> Type` is a _higher-order_ type constructor (it takes a unary type constructor and yields a type).

### Type application

A type constructor is applied to a type or another type constructor, depending on its kind, similarly to how functions are applied to arguments at the term level. `C T` applies the type constructor `C` to the type `T`. Type application associates to the left, so the type `A B C` is the same as the type `(A B) C`.

### Function types

The type `X -> Y` is a type for functions that take arguments of type `X` and yield results of type `Y`. Application of the binary type constructor `->` associates to the right, so the type `X -> Y -> Z` is the same as the type `X -> (Y -> Z)`.

### Tuple types

The type `(A,B)` is a type for binary tuples (pairs) of values, one of type `A` and another of type `B`.  The type `(A,B,C)` is a triple, and so on.

The type `(A)` is the same as the type `A` and is not considered a tuple.

The nullary tuple type `()` is the type of the unique value also written `()` and is pronouced “unit”.

In the standard Unison syntax, tuples of arity 2 and higher are actually of a type `Tuple a b` for some types `a` and `b`. For example, `(X,Y)` is syntactic shorthand for the type `Tuple X (Tuple Y ())`.

Tuples are either constructed with the syntactic shorthand `(a,b)` (see [tuple literals](#literals)) or with the built-in `Tuple.Cons` data constructor: `Tuple.Cons a (Tuple.Cons b ())`.

### Built-in types

Unison provides the following built-in types:

* `.base.Nat` is the type of 64-bit natural numbers, also known as unsigned integers. They range from 0 to 18,446,744,073,709,551,615.
* `.base.Int` is the type of 64-bit signed integers. They range from -9,223,372,036,854,775,808 to +9,223,372,036,854,775,807.
* `.base.Float` is the type of [IEEE 754-1985](https://en.wikipedia.org/wiki/IEEE_754-1985) double-precision floating point numbers.
* `.base.Boolean` is the type of Boolean expressions whose value is `true` or `false`.
* `.base.Bytes` is the type of arbitrary-length 8-bit byte sequences.
* `.base.Text` is the type of arbitrary-length strings of Unicode text.
* `.base.Char` is the type of a single Unicode character.
* The trivial type `()` (pronounced “unit”) is the type of the nullary tuple. There is a single data constructor of type `()` and it’s also written `()`.

See [literals](#literals) for more on how values of some of these types are constructed.

### Built-in type constructors

Unison has the following built-in type constructors.

* `(->)` is the constructor of function types. A type `X -> Y` is the type of functions from `X` to `Y`.
* `base.Tuple` is the constructor of tuple types. See [tuple types](#tuple-types) for details on tuples.
* `.base.List` is the constructor of list types. A type `List T` is the type of arbitrary-length sequences of values of type `T`. The type `[T]` is an alias for `List T`.
* `.base.Request` is the constructor of requests for abilities. A type `Request A T` is the type of values received by ability handlers for the ability `A` where current continuation requires a value of type `T`.

### User-defined types

New types can be declared as described in detail in the [User-defined types](#user-defined-data-types) section. These include ordinary [data types](#user-defined-data-types), [unique types](#unique-types), and [record types](#record-types). A type declaration introduces a _type_, a corresponding _type constructor_, one or more _data constructors_ that (collectively) construct all possible values of the type, and (in the case of record types) accessors for the named arguments of the type's single data constructor.

<a id="abilities"></a>

## Abilities and ability handlers

Unison provides a convenient feature called _abilities_ which lets you use the same ordinary Unison syntax for programs that do (asynchronous) I/O, stream processing, exception handling, parsing, distributed computation, and lots more.  Unison's system of abilities (often called "algebraic effects" in the literature) is based on [the Frank language by Sam Lindley, Conor McBride, and Craig McLaughlin](https://arxiv.org/pdf/1611.09259.pdf). Unison diverges slightly from the scheme detailed in this paper. In particular:

* Unison's ability polymorphism is provided by ordinary polymorphic types, and a Unison type with an empty ability set explicitly disallows any abilities. In Frank, the empty ability set implies an ability-polymorphic type.
* Unison doesn't overload function application syntax to do ability handling; instead it has a separate [`handle` construct](#ability-handlers) for this purpose.

### Abilities in function types

The general form for a function type in Unison is `I ->{A} O`, where `I` is the input type of the function, `O` is the output type, and `A` is the set of _ability requirements_ of the function. More generally, this can be any comma-separated list of types, like `I ->{A1,A2,A3} O`.

A function type in Unison like `A -> B` is really syntactic sugar for a type `A ->{e} B` where `e` is some set of abilities, possibly empty. A function that definitely requires no abilities has a type like `A ->{} B` (it has an empty set of abilities).

<a id="ability-typechecking"></a>

### The typechecking rule for abilities

The general typechecking rule used for abilities is this: calls to functions requiring abilities `{A1,A2}` must be in a context where _at least_ the abilities `{A1,A2}` are _available_, otherwise the typechecker will complain with an ability check failure. Abilities can be made available using a `handle` block (discussed below) or with a type signature: so for instance, within the body of a function `Text ->{IO} Nat`, `{IO}` is available and therefore:

* We can call a function `f : Nat ->{} Nat`, since the ability requirements of f are `{}` which is a subset the available `{IO}` abilities.
* We can also call a function `g : Text ->{IO} ()`, since the requirements of `g` are `{IO}` which is a subset of the available `{IO}` abilities.

For functions accepting multiple arguments, it might seem we need a different rule, but the rule is the same: the body of the function can access the abilities attached to the corresponding function type in the signature. Here's an example that won't typecheck, since the function body must be pure according to the signature:

```unison
doesNotWork : Text ->{IO} Text ->{} Nat
doesNotWork arg1 arg2 =
  printLine "Does not work!"
  42
```

However, if we do the `IO` before returning the pure function, it typechecks just fine:

```unison
doesWork : Text ->{IO} Text ->{} Nat
doesWork arg1 =
  printLine "Works great!"
  (arg2 -> 42) -- we return a pure `Text ->{} Nat`
```

For top-level definitions which aren't contained in a function body, _they are required to be pure_. For instance, this doesn't typecheck:

```unison
msg = printLine "hello"
```

But if we say `msg = '(printLine "Hello")` that typechecks fine, since the `printLine` is now inside a function body whose requirements inferred as `{IO}` (try it!).

> 🤓 This restriction on top level definitions needing to be pure might lifted in a future version of Unison.

#### Ability inference

When inferring ability requirements for `f`, the ability requirements become the union of all requirements for functions that can be called within the body of the function.

### User-defined abilities

A user-defined ability is declared with an `ability` declaration such as:

``` unison
ability Store v where
  get : v
  put : v -> ()
```

This results in a new ability type constructor `Store` which takes a type argument `v`. It also create two value-level constructors named `get` and `put`. The idea is that `get` provides the ability to "get" a value of type `v` from somewhere, and `put` allows "putting" a value of type `v` somewhere. Where exactly these values of type `v` will be kept depends on the handler.

The `Store` constructors `get` and `put` have the following types:

* `get : forall v. {Store v} v`
* `put : forall v. v ->{Store v} ()`

The type `{Store v}` means that the computation which results in that type requires a `Store v` ability and cannot be executed except in the context of an _ability handler_ that provides the ability.

<a id="handlers"></a>

### Ability handlers

A constructor `{A} T` for some ability `A` and some type `T`  (or a function which uses such a constructor), can only be used in a scope where the ability `A` is provided. Abilities are provided by `handle` expressions:

``` unison
handle e with h
```

This expression gives `e` access to abilities handled by the function `h`. Specifically, if `e` has type `{A} T` and `h` has type `Request A T -> R`, then `handle e with h` has type `R`. The type constructor `Request` is a special builtin provided by Unison which will pass arguments of type `Request A T` to a handler for the ability `A`. Note that `h` _must_ accept an argument of type `Request A T` to handle `e` of type `{A} T`&mdash;ultimately, a type error will result if an ability is required in a scope where it is not provided by an enclosing handle expression.

The examples in the next section should help clarify how ability handlers work.

### Pattern matching on ability constructors

Each constructor of an ability corresponds with a _pattern_ that can be used for pattern matching in ability handlers. The general form of such a pattern is:

``` unison
{A.c p_1 p_2 p_n -> k}
```

Where `A` is the name of the ability, `c` is the name of the constructor, `p_1` through `p_n` are patterns matching the arguments to the constructor, and `k` is a _continuation_ for the program. If the value matching the pattern has type `Request A T` and the constructor of that value had type `X ->{A} Y`, then `k` has type `Y -> {A} T`.

The continuation will always be a function accepting the return value of the ability constructor, and the body of this function is the remainder of the `handle .. with` block immediately following the call to the constructor. See below for an example.

A handler can choose to call the continuation or not, or to call it multiple times. For example, a handler can ignore the continuation in order to handle an ability that aborts the execution of the program:

``` unison
ability Abort where
  aborting : ()

-- Returns `a` immediately if the program `e` calls `abort`
abortHandler : a -> Request Abort a -> a
abortHandler a = cases
   { Abort.aborting -> _ } -> a
   { x } -> x

p : Nat
p = handle
      x = 4
      Abort.aborting
      x + 2
    with abortHandler 0

```

The program `p` evaluates to `0`. If we remove the `Abort.aborting` call, it evaluates to `6`.

Note that although the ability constructor is given the signature `aborting : ()`, its actual type is `{Abort} ()`.

The pattern `{ Abort.aborting -> _ }` matches when the `Abort.aborting` call in `p` occurs. This pattern ignores its continuation since it will not invoke it (which is how it aborts the program). The continuation at this point is the expression `_ -> x + 2`.

The pattern `{ x }` matches the case where the computation is pure (makes no further requests for the `Abort` ability and the continuation is empty). A pattern match on a `Request` is not complete unless this case is handled.

When a handler calls the continuation, it needs describe how the ability is provided in the continuation of the program, usually with a recursive call, like this:

``` unison
use .base Request

ability Store v where
  get : v
  put : v -> ()

storeHandler : v -> Request (Store v) a -> a
storeHandler storedValue = cases
  {Store.get -> k} ->
    handle k storedValue with storeHandler storedValue
  {Store.put v -> k} ->
    handle k () with storeHandler v
  {a} -> a
```

Note that the `storeHandler` has a `with` clause that uses `storeHandler` itself to handle the `Request`s made by the continuation. So it’s a recursive definition. The initial "stored value" of type `v` is given to the handler in its argument named `storedValue`, and the changing value is captured by the fact that different values are passed to each recursive invocation of the handler.

In the pattern for `Store.get`, the continuation `k` expects a `v`, since the return type of `get` is `v`. In the pattern for `Store.put`, the continuation `k` expects `()`, which is the return type of `put`.

It's worth noting that this is a mutual recursion between `storeHandler` and the various continuations (all named `k`). This is no cause for concern, as they call each other in tail position and the Unison compiler performs [tail call elimination](#function-application).

An example use of the above handler:

``` unison
modifyStore : (v -> v) ->{Store v} ()
modifyStore f =
  v = Store.get
  Store.put (f v)
```

Here, when the handler receives `Store.get`, the continuation is `v -> Store.put (f v)`. When the handler receives `Store.put`, the continuation is `_ -> ()`.

## Use clauses

A _use clause_ tells Unison to allow [identifiers](#identifiers) from a given [namespace](#namespace-qualified-identifiers) to be used [unqualified](#identifiers) in the lexical scope where the use clause appears.

In this example, the `use .base.List` clause allows the definition that follows it to refer to `.base.List.take` as simply `take`:

``` unison
use .base.List

oneTwo = take 2 [1,2,3]
```

The general form of `use` clauses is as follows:

``` unison
use namespace name_1 name_2 .. name_n
```

Where `namespace` is the namespace from which we want to use names unqualified, and `name_1` through `name_n` are the names we want to use. If no names are given in the `use` clause, Unison allows all the names from the namespace to be used unqualified. There's no performance penalty for this, as `use` clauses are purely a syntactic convenience. When rendering code as text, Unison will insert precise `use` clauses that mention exactly the names it uses, even if the programmer omitted the list of names.

See the section on [identifiers](#identifiers) for more on namespaces as well as qualified and unqualified names.
