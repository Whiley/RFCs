- Feature Name: `remove-any-neg`
- Start Date: `27-10-2017`
- RFC PR: https://github.com/Whiley/RFCs/pull/20
- See Also: [RFC#0017](https://github.com/Whiley/RFCs/blob/master/text/0017-runtime-type-information.md)
- Tracking Issue: [#827](https://github.com/Whiley/WhileyCompiler/issues/827)

# Summary

This proposes a simplification of the source-level syntax for Whiley.
Specifically, the removal of the primitive `any` type and negation
types and the addition of (internal) difference types.

# Motivation

The presence of the so-called _infinite types_ presents some
significant challenges for the compilation of Whiley to low-level
platforms (e.g. C).  Recalling from
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

Reducing the number of cases to be considered would improve progress
on implementing a wider range of backends.  _It should be noted,
furthermore, that the removal of such types does not preclude their
reintroduction at a later date._

This proposal aims to remove the `any` type and negation types
(e.g. `!int`) from the language.  Indeed, the `any` type is rarely
used in practice and its removal will have minimal effect.  In
contrast, negation types are used extensively for flow typing and,
hence, an alternative is presented here.

# Technical Details

The `any` type is the _top_ type in Whiley's type system and has
  numerous representations.  For example, `any|any`, `int|!int`,
  `!(!int&int)`, `!(int&bool)` are all equivalent to `any`.  Thus, the
  following currently compiles:

```Whiley
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

```Whiley
type record is { void f }
```

This produces the following error message:

```
./test.whiley:1: empty type encountered
type record is { void f }
               ^^^^^^^^^^
```

Likewise, this equivalent form also does not compile:

```Whiley
type record is { (int&!int) f }
```

## Negation Types

The removal of negation types is more involved because of their
involvement in flow typing.  The following illustrates the simplest
example:

```Whiley
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
is `((int|null)&!int)`.  For a type test `x is T` where `x` has
declared type `S` the type of `x` on the false branch is, in general,
`S&!T`.  This type is a representation of the _difference type_ `S -
T`.

_The downside with general negation types is that they provide more
functionality than actually required_.  For example, we want to
support `S&!T` but we also support `S|!T`.  This is problematic as it
enables alternative representations of `any`, such as `int|!int`.
This proposal **removes negation types from the Whiley language** and
replaces them with _internal_ types of the form `S-T`.  Here, internal
means they cannot be expressed at the source level (though,
potentially, we might lift this restriction later).  This ensures that
flow typing is still fully supported as before, but removes a number
of complex issues regarding types.

## Impact

The impact of removing `any` and negation types is reasonably
profound.  We can make the following strong(ish) statement:

**OBSERVATION:** _Any type `T` not containing an open record can be
  reduced to a (potentially simpler) type `S` which does not contain
  intersection and/or difference types._

This observation is basically stating that we can always reduce a
general type (ignoring open records for now) to one which only uses
the union connective.  Some examples:

* `(int|null)&int` gives `int`
* `(int|null)-int` gives `null`
* `{int|null f}&{int f}` gives `{int f}`
* `{int|null f}-{int f}` gives `{null f}`
* `(int|null|bool)-int` gives `int|bool`

Recursive types are not themselves a problem here.  For example,
consider these recursive types:

```Whiley
type IntList is null | { IntList next, int data }
type IntBoolList is null | { IntBoolList next, int|bool data}
```

Then, the type `IntList&IntBoolList` reduces to `IntList`.  Likewise,
`{null next, int|null data}&IntList` reduces to `{null next, int
data}`.  Of course, the algorithm for performing this simplification
remains somewhat involved.

Finally, the question of _open records_ remains.  For example, the
type `{int x, ...} - {int x, int y}` cannot be further reduced.  This
proposal does not address this issue, though it may be address in a
subsequent RFC.


# Terminology

* **Difference Type**.  A type of the form `T-S` represents the set of
elements represented by `T` less the set represented by `S`
(i.e. it corresponds to set difference).  For example,
`(int|null)-int` can be reduced to `int`.

# Drawbacks and Limitations

A small amount of existing code will be broken.

# Unresolved Issues

None.

