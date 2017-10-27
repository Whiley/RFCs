- Feature Name: `remove-any-neg`
- Start Date: `27-10-2017`
- RFC PR: (leave this empty)
- See Also: [RFC#0017](https://github.com/Whiley/RFCs/blob/master/text/0017-runtime-type-information.md)

# Summary

This proposes a simplification of the source-level syntax for Whiley.
Specifically, the removal of the primitive `any` type and negation
types from the source-level syntax of Whiley.

# Motivation

The presence of the so-called infinite types presents some significant
difficulties for the compilation of Whiley to low-level platforms,
such as C.  Recalling from
[RFC#0017](https://github.com/Whiley/RFCs/blob/master/text/0017-runtime-type-information.md),
the infinite types are defined as follows:

- **Any**.  The simplest kind of infinite type is one involving `any`.
  For example, `any`, `{any f}`, `any[]` are all infinite types.

- **Negations**.  The next simplest kind of infinite type is one
  involving a negation.  With the exception of `!any`, all true
  negations are infinite types.  For example, `!int`, `!{int f}`,
  `!(int[])`, are all infinite types.

- **Open Records**.  A surprising example of an infinite type is the
  open record.  For example, `{int f, ...}` contains an infinite
  number of subtypes, including `{int f}`, `{int f, int g}`, `{int f,
  int g, int h}`, etc.

- **Recursive Types**.  Another source of infinite types are recursive
  types.  In fact, every recursive type is infinite.  For example,
  type `List` defined as `null | { List next, int data}` contains an
  infinite number of subtypes, including `null`, `{null next, int
  data}`, `{ {null next, int data}, int data}`, etc.

Reducing the number of cases to be considered here would improve
progress on implementing a wider range of back-ends.  It should be
noted, furthermore, that the removal of such types does not preclude
their reintroduction at a later date.

This proposal aims to remove the `any` type and negations types
(e.g. `!int`) from the language.  Indeed, the `any` type is rarely
used in practice and its removal will minimal effect.  In contrast,
negation types are used extensively for flow typing and, hence, an
alternative is presented here.

# Technical Details

The `any` type is the _top_ type in Whiley's type system and has
  numerous representations.  For example, `any|any`, `int|!int`,
  `!(!int&int)`, `!(int&bool)` are all equivalent to `any`.  Thus, the
  following currently compiles:

```
function id(any x) -> int|!int:
	return x
```

The removal of the `any` type itself is straightforward.  However, the
  removal of those equivalent representations is more involved.  Since
  `void` is already prohibited, we can immediately rule out `!void`.
  Likewise, types equivalent to `void` are also prohibited (see
  below).  Therefore, we need only prevent types of the form
  `int|!int`.  The simplest way of doing this it to remove negation
  types altogether (see below).


## Void Types

The removal of types equivalent to `void` is already required and
implemented.  For example, the following fails to compile:

```
type record is { void f }
```

This produces the following error message:

```
./test.whiley:1: empty type encountered
type record is { void f }
               ^^^^^^^^^^
```

Likewise, this equivalent form also does not compile:

```
type record is { (int&!int) f }
```

## Negation Types

The removal of negation types is more involved because of their
involvement in flow typing.  The following illustrates the simplest
example:

```
function f(int|null x) -> int:
    if x is int:
	   return x
    else:
	   return x
```

This fails to compile with the following error:

```
./test.whiley:5: expected type int, found ((int|null)&!int)
	   return x
	          ^
```

Here, we can clearly see the type computed for `x` on the false branch
is `((int|null)&!int)`.  For a type test `x is T` where
`x` has declared type `S` the type of `x` on the false branch is, in
general, `S&!T`.  This type is a representation of the difference type
`S - T`.

# Terminology

# Drawbacks and Limitations

# Unresolved Issues

