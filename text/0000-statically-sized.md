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
the programmer to dictate in absolute terms what representation
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
  [this paper](http://homepages.ecs.vuw.ac.nz/~djp/files/SEUS15.pdf)).
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
  and simple mechanism for specifying representation.  Finally, using
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
  `|arr| == n`.
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
bytes of unknown length.  The key is that denoting the array as having
unknown length affects its underlying representation.  Specifically,
no length field is stored with the array.  Thus, it can be represented
as just a sequence of zero or more elements to match the C representation.

The above illustrates the use of the array length operator on an
unknown-length array in a specification element (i.e. `|items|`).  By
allowing this operator in specification elements, we can still reason
about the array's actual length in an abstract sense.

Finally, as an example to illustrate, we can provide the following
mapping from C99 fixed-width data types:

- `int8_t` => `int:8`, `int16_t` => `int:16`, `int32_t` => `int:32`,
etc
- `uint8_t` => `uint:8`, `uint16_t` => `uint:16`, `uint32_t` => `uint:32`, etc
- `size_t` => usize
- `char` => `uint:8`

Similar mappings exist for other languages.  For example, a Java `int`
maps to `int:32` whilst a `long` maps to `int:64`, etc.  Likewise, a
Java `int[]` maps to a `int:32[]`, etc.

## Coercions

This proposal treats the status of a type (i.e. statically sized or
dynamically sized) as a _type modifier_ rather than as a _type_.  This
has significant implications for the meaning of such types.  For
example, a variable of type `int:32` can flow into a variable of type
`int:8` _without the need for an explicit cast_.  The following
illustrates a valid Whiley function under this proposal:

```
function abs(int:32 x) -> (uint:32 r)
ensures r >= 0:
   //
   if x >= 0:
      return x
   else:
      return -x
```

From a typing perspective, this function allows a variable of type
`int:32` to flow into one of type `uint:32`.  This is clearly unsound
in an unrestricted fashion.  For example, `-1` is a valid member of
type `int:32` but not `uint:32`.  Nevertheless, the above program is
considered valid under this proposal.

The reason the above program is considered valid is that, under
verification, the constraints imposed by the different integer
representations are shown to hold.  **In other words, validity of the
underlying representation depends upon verification.**

As one exception, we note that `int[?]` cannot flow into `int[]` (with
or without a cast).  This is simply because it is logically impossible
to implement as, at the point we have an instance of `int[?]` _we no
longer have a length variable_.  However, it is valid for a variable
`arr` of type `int[?]` to flow into a variable of type `int[n]` (for
some `n`).  The safety of this relies on the verifier to establish
that `|arr| == n`.

## Typing

An important question is how expressions involving statically-sized
data types will be typed.  For example, consider this Whiley program:

```
function sum(int:16 x) -> (int:32 y):
   return x + 1
```

What type should be inferred for the expression `x+1`?  There are two
possible approaches:

- **Forward Propagation.** In this approach, the type of an expression
  is inferred by determining the worse-case bound on its output.  In
  the above example, the inferred type would `int:17`.
- **Backward Propagation.** In this approach, the type of an
  expression is inferred by from the target location.  That is, the
  type of the variable into which it will flow.  In the above example,
  the inferred type would `int:32` as this follows from the return type.

_This proposal does not currently mandate which of these approaches
should be taken_.  This is because, at this stage, it is unclear what
effect either of these will have.

Some guidelines regarding typing of expressions (see
[this paper](http://homepages.ecs.vuw.ac.nz/~djp/files/SEUS15.pdf) for
more details):

- `operator+(int:n,int m)` produces a result of type `int:(n+m)`.
- `operator-(int:n,int m)` produces a result of type `int:(n+m)`.
- `operator*(int:n,int m)` produces a result of type `int:(n*m)`.
- `operator/(int:n,int m)` produces a result of type `int:n`.
- `operator%(int:n,int m)` produces a result of type `int:m`.
- `operator||(T[])` produces a result of type `usize`.
- `operator[](T[],usize)` produces a result of type `T`.

Further details regarding the typing of operators will need to be
worked out.  We note the above implies the operands of an operator are
not required to have the same static size, as this would be quite
unwieldy.

## Verification

Whilst coercions between different statically-sized types happen
implicitly at the type level, safety constraints are imposed during
verification.  We now clarify the constraints imposed, where `x^y`
is taken to mean `x` to the _power of_ `y` (e.g. `2^4` gives `16`).

- **Signed Integers**.  Values of a signed integer `int:n` should be
  in the range `-2^(n-1)` to `2^(n-1)-1`.

- **Unsigned Integers**.  Values of an unsigned integer `uint` should
be in the range `0` to `2^n`.

For a more detailed explanation of these constraints, see the
Wikipedia page on
[Two's Complement](https://en.wikipedia.org/wiki/Two%27s_complement).
As an example, the following program should verify:

```
function f(int:8 x) -> (int:4 r):
   if x >= 0 && x <= 7:
      return x
   else:
      return 0
```

This verifies primarily because `7 <= 2^(4-1)-1` reduces to `7 <= 7`
which clearly holds.  In contrast, the following does not verify:

```
function f(int:8 x) -> (int:4 r):
   return x
```

This fails to verify because `2^(8-1)-1 <= 2^(4-1)-1` reduces to `127 <= 7`
which clearly does not hold.

# Terminology

- **Statically-Sized Type**.  A statically-sized type is one whose
representation size (in bits) is known at compile time.

- **Dynamically-Sized Type**.  A dynamically-sized type is one whose
representation size (in bits) is unknown at compile time.

- **Fixed-Length Array**.  A fixed-length array is one whose length (in
elements) is known at compile time.

- **Unknown-Length Array**.  An unknown-length array is one whose
  length is unknown either at compile time or runtime.

# Drawbacks and Limitations

- **Implicit Coercions**.  The flexibility offered by implicit
  coercions between types of different size presents a potential
  hazard.  In particular, for programs which are not verified, there
  is a potential (and silent) _loss of information_.

# Unresolved Issues

- Constraints on `usize`.  At this time, it's unclear how one can
  safely verify programs involving types of `usize` as there are no
  implicit bounds placed on them.  For the case of a variable
  (e.g. `i`) being used as an index into an array (e.g. `arr`) it is
  relatively straightforward as we already require `i < |arr|`.
  Hence, `i` must fit safetly into the type `usize`.  However,
  consider an expression of the form `x < |arr|` where `x` has some
  statically sized type `int:n`.  _Should we cast `x` to type `usize`
  for the comparison?_ This seems questionable since we could lose
  information.  A clever compiler could spot this and return `false`
  for the condition, but this is unclear.

