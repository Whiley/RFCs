- Feature Name: statically_sized_types
- Start Date: 22-05-2017

# Summary

This proposal is for introducing _statically sized types_ to the
Whiley language.  Such types occupy a fixed (and known) amount of
space at runtime.  For example, a statically sized integer type might
occupy exactly 32bits at runtime.

# Motivation

The Whiley Language currently supports dynamically-sized data types.
For example, the `int` type in Whiley is unbound and can
hold any possible integer (theoretically, at least).  Such types are
naturally difficult to represent efficiently on a machine.
Furthermore, they impede communication between Whiley and foreign
code which employs fixed-sized data types (e.g. `int32` in C).

This proposal addresses the problem of specifying what underlying
representation a given type should have.  That is, providing a way for
the programming to dictate in absolute terms what representation
should be used.  This allows the programmer to optimise the underlying
representation as he/she sees fit.  It also enables them to interface
with foreign code more easily.

Several alternatives that could be considered to this proposal:

- **Library definitions of statically sized types**.  The standard
  library currently includes various definitions of statically sized
  types (e.g. `u8`,`i16`,`u32`, etc).  Knowledge of these could be
  hard-coded into the compiler.  However, this unnecessarily couples
  the compiler with the standard library.  Furthermore, there are
  certain foreign data types (e.g. C strings) which cannot be easily
  modelled in this way.

- **Range Analysis**.  Another approach previously explored was the
  use of integer range analysis (see
  [here](http://homepages.ecs.vuw.ac.nz/~djp/files/SEUS15.pdf)).
  Here, the compiler examines the invariants given on a datatype to
  determine an appropriate representation.  For example, consider a
  type `(int n)` where `n >= 0 && n <= 255`.  The range returned for
  this type would be `0 .. 255` and, hence, it could fit into a single
  `byte`.  This approach improves upon the previous in that it does
  not couple the compiler with the standard library.  However, it
  still cannot encode certain data types (e.g. C strings).
  Furthermore, the range analysis process is hard to state clearly
  because of the complex nature of invariants in Whiley.  For example,
  what range should we return for the constraint `n >= 0 && (n <= 255
  || n < 128)`?  Probably `0 .. 255` makes the most sense here, but
  invariant expressions can be arbitrarily complex and we need a clear
  and simple mechanism for specifying representation.  Finally, use
  range analysis makes interoperation with foreign code more
  challenging.  This is because we may have foreign data types whose
  representation is not minimal for their true invariants.  For
  example, an `int` parameter with only seven possible values
  (e.g. `1`..`7`).  In such case, we need to separate the
  representation from the constraints.

As a result of their limitations, the above approaches are not
considered suitable for addressing the problem.

# Technical Details

This proposal introduces the following data types which are referred
to as _statically sized_:

- **Signed Integers**. The type of statically-sized signed integers is
  `int:n`, where `n` is a numeric constant denoting the number of bits
  of precision used for a twos-compliment representation (typically
  `8`, `16`, `32`, etc).
- **Unsigned Integers**. The type of statically-sized unsigned
  integers is `uint:n`, where `n` is a numeric constant denoting the
  number of bits of precision used (typically `8`, `16`, `32`, etc).
  In addition, the special type `usize` denotes a fixed-width unsigned
  integer sufficient to the hold the length of an array.
- **Arrays**. The type of _fixed-length_ arrays is given by `T[n]`
  where `n` is an unsigned numeric constant.  A fixed-length array is
  statically sized if its element type `T` is statically sized.  For a
  fixed-length array `arr` of type `T[n]`, it is implicit that
  `|arr|==n`.
- **Unknown-Length Arrays**. The type of _unknown-length_ arrays is
  given by `T[?]`.  Such arrays have a maximum length, but this is
  unknown and the _array length operator_ is defined for such types
  only within specification elements.

In addition, the existing types `int` and `T[]` are referred to as
_dynamically sized_.  In contrast, the existing types `bool`, `null`,
`&T` and function/method types are referred to as _statically sized_.
The status of other existing compound types (e.g. records, unions,
intersections, negations) is determined by their element types.
Specifically, if all element types are statically-sized then the
enclosing type is statically sized.  For example, `{ int:8 x, int:8
y}` is statically sized, whilst `int|bool` is not.

## Foreign Functions

The proposed types allow one to specify foreign functions more
precisely.  For example, the well-known `strncpy` function has the
following type in C:

```
char *strncpy(char *, const char *, size_t);
```

Using the above types, we can give a corresponding Whiley type for
this:

```
type C_string is (uint:8[?] items)
// C strings are null terminated
where some { i in 0..|items| | items[i] == NULL }

&C_string strncpy(&C_string , &C_string, usize)
```

Here, the type `&(uint:8[?])` denotes a reference to an array of
bytes of unknown length.  The key is that denoting the array as
unknown length affects its underlying representation.  Specifically,
no length field is stored with the array.  Thus, it can be represented
as just a sequence of zero or more elements to match the C representation.

The above illustrates the use of the array length operator on an
unknown-length array in a specification element (i.e. `|items|`).  By
allowing this operator in specification elements, we enable the
ability to reason about the array's actual length in an abstract
sense.

Finally, as an example to illustrate, we can provide the following
mapping from C99 fixed-width data types:

- `int8_t` => `int:8`, `int16_t` => `int:16`, `int32_t` => `int:32`,
etc
- `uint8_t` => `uint:8`, `uint16_t` => `uint:16`, `uint32_t` => `uint:32`, etc
- `size_t` => usize
- `char` => `uint:8`

Similar mappings exist for other languages.  For example, a Java `int`
maps to `int:32` whilst a `long` maps to `int:64`, etc.

## Implicit Coercions

## Verification

The subtype relationship between types operates in roughly the
expected fashion.  Specifically:

- **(Rule Int-StaticDynamic)** `int:n` subtypes `int` for all `n`.
  Likewise, `uint:n` subtypes `uint` for all `n`.
- **(Rule Int-StaticStatic)** `int:n` subtypes `int:m` if `n <= m`.
Likewise, `uint:n` subtypes `uint:m` if `n <= m`.
- **(Rule Int-SignedUnsigned)**.

_Do we need subtyping rules?  Perhaps these are more guidelines for
information loss?_

# Notes

Array length operator returns `usize`.

Does fixed length array actually have specified length?

# Terminology

In changing the language, it is important to develop a suitable
"vernacular" for talking about the new aspects of the language.  For
example, if a new piece of syntax is proposed, then a standard way of
referring to this should be proposed.  Likewise, if a new phase of the
compiler is introduced, then suitable terminology for discussing the
various aspects of this should be developed.

# Drawbacks and Limitations

This should identify any potential drawbacks or limitations of the
proposed change.  For example, if the proposed change would break
existing code, this should be clearly identified.  Likewise, if the
change could impact upon other proposed changes, this should be
identified.

# Unresolved Issues

List any currently unresolved aspects of the change proposal.  These
will need to be adequately resolved before the RFC is accepted.
Identifying these issues provides a way for the author(s) of the RFC
to leverage the community in finding appropriate solutions.