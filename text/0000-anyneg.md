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

**Any.** The `any` type is the _top_ type in Whiley's type system and
  has numerous representations.  For example, `any|any`, `int|!int`,
  `!(!int&int)`, `!(int&bool)` are all equivalent to `any`.  The
  removal of the `any` type itself is straightforward.  However, the
  removal of those equivalent representations is more involved.  Since
  `void` is already prohibited, we can immediately rule out `!void`.
  However, we ideally should rule out other types equivalent to `void`
  (see below).  Unfortunately, the removal of types equivalent to
  `void` is insufficient.  We must also prevent types of the form
  `int|!int`.  The simplest way of doing this it to remove negation
  types (see below).
  
# Terminology

# Drawbacks and Limitations

# Unresolved Issues

