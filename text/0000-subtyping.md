- Feature Name: `simple_subtyping`
- Start Date: `23-07-2019`
- RFC PR: (leave this empty)

# Summary

Subtyping in Whiley is currently complex and challenging, leading to
numerous implementation issues.  Furthermore, there are significant
problems related to type testing.

# Motivation

Subtyping in Whiley is currently based around the concept of
separating out the _underlying type_, whilst leaving the verifier to
pick up the pieces.  For example, consider this snippet:

```
type nat is (int n) where n >= 0

function abs(int x) -> (nat r):
  if x >= 0:
     return x
  else:
     return -x
```

This nicely illustrates the motivation behind the current approach to
subtyping in Whiley.  Specifically, there is no need to cast `x` at
any point.  To make this work, the compiler reasons that the
underlying type of `x` (i.e. `int`) is a subtype of the underlying
type of `nat` (also `int`).  It then defers checking the `if`
condition meets the `nat` condition to the verifier.

A key underlying principle of the above is that, even without the
verifier, we can using runtime checking when the implicit coercions
arise.  **However, there be dragons here.**

### Callables

The most obvious scenario where this approach fails is with `function` or
`method` types.  For example, consider this:

```
type fun_t is function(int)->(nat)

function id(int x) -> (int r):
   return x

function f() -> fun_t:
   return &id
```

This currently compiles but, in fact, is not type safe.  Furthermore,
there is no way for us to check this at runtime.

### References

References are another similar situation to the above.  Consider this:

```
function id(&int x) -> (&nat r):
   return x
```

Again, this compiles but is not type safe.  Furthermore, whilst this
can be partially checked at runtime (e.g. by checking `*r` is `nat` at
the point of return), it remains fraught with danger.  For example,
the following illustrates:

```
method main():
   &int x = new 0
   &nat y = id(x)
   // 
   *x = -1
```

Again, this is legitamate code which does compile but whose invariants
cannot easily be checked at runtime (though can be checked by the
verifier).

### Type Tests

Runtime type tests in Whiley represent another whole class of
problems.  For a type test `x is T` where `x` has type `S`, the only
current requirement is that `T&S != 0` for the type test to be
considered valid.  Some problematic examples:

```
type fnat_t is function(nat)->(int)
type fint_t is function(int)->(int)

function f(fint_t|null xs) -> (int r):
   if xs is fnat_t:
      return 1
   else:
      return 0
```

This example is particularly problematic because it must be possible
to evaluate a type test!  Likewise, a similar example for references
can easily be constructed.

Another problematic scenario with runtime type tests is the extraction
of a particular type from an open record.  The following illustrates:

```
type Point is { int x, int y, ... }

function getZ(Point p) -> (int r):
   if p is { int x, int y, int z }:
      return z
   else:
      return 0
```

This is difficult to implement on some platforms as it requires that
open records carry runtime type information about the fields they
contain.

Finally, an interesting aspect of runtime type tests as currently
implemented is that they lose precision in some cases.  For example,
consider this:

```
type bnat is (nat|bool x)

function check(bnat x) -> (bool r):
   if x is nat:
      return false
   else:
      return x
```

Perhaps surprisingly, this fails to compile and reports the following error:


```
test.whiley:7: expected type bool, found bnat-nat
      return x
             ^
```

The issue is that the type system is conservative in the treatment of
any constrained type.

# Technical Details

This proposal aims to resolve the above issues by simplifying the way
that subtyping is implemented in Whiley.  Whilst this may result in
some syntactic overhead, it will result in a safer and more
well-defined language.  Furthermore, we will retain the ability to
fall back on the verifier for guaranteeing certain properties.

### Strict Subtyping

The proposed subtyping operator is strict about the handling of type
invariants.  Specifically, it will only allow types to be considered
subtypes when it is certain they are.  Consider the following:

```
type nat is (int x) where x >= 0
type pos1 is (int x) where x > 0
type pos2 is (nat x) where x > 0
```

In the current approach, the above are all currently considered
subtypes of each other (modulo verification).  In the proposed scheme,
it follows that `nat <: int`, `pos1 <: int`, `pos2 <: int` and also
that `pos2 <: nat`.  However, it does not follow that `nat <: pos1` or
`pos1 <: pos2`, etc.  **As a special exception, nominal types without
invariants are always unified with their underlying type.** Thus, for
example, if we have `type bnat is nat` when it follows that `pos2 <:
bnat`, etc.  Subtyping for references and function types is also
strict.  Finally, subtyping between open records and closed records is
not permitted.

### Casting

To work around this stricter notion of subtyping, casting will be
employed.  Thus, the `abs()` example from above becomes:

```
function abs(int x) -> (nat r):
  if x >= 0:
     return (nat) x
  else:
     return (nat) -x
```

Casting is used to indicate an unchecked property requires
verification to be type safe.  Casting may also be used to signal
certain coercions are required (e.g. from one function type to
another, or from a closed record to an open record).

### Runtime Type Tests

For a runtime type test `x is T` where `x` has type `S` we require
that `T` is a subtype of `S`.  This means, for example, that the
following example now compiles:

```
type bnat is (nat|bool x)

function check(bnat x) -> (bool r):
   if x is nat:
      return false
   else:
      return x
```

The type system now correctly concludes that `x` has type bool at the
`return` statement.  Treatment of type selection is made as precise as
possible.  A type `T` _strictly subtypes_ another type `S` when `T <:
S` and. furthermore, `T` is constrained (i.e. has an invariant).
Thus, `S - T = 0` only when `S` is _equivalent_ to `T` (i.e. when `S
<: T` and `T <: S`).

# Terminology

* _Underlying Type_.  Given a declaration `type T is (S x)`, we say
  that `S` is the _underlying type_ of `T`.

Observe that we are reusing the old terminology here, though that was
never well defined in the first place.

# Limitations

Some simple programs Whiley may no longer compile.  The following
illustrates:

```
type Point is {int x, int y} where x > y

function f(int x, int y) -> Point:
   return {x,y}
```

In principle, this should not compile because `{int x, int y} <:
Point` no longer holds.  However, forward type inference could be used
to resolve this case.  A similar example would be:

```
function f(int x, int y) -> Point:
   {int x, int y} r = {x,y}
   return r
```

This probably cannot be resolved.  Therefore, we can either employ a
cast or reintroduce named records (e.g. `Point{x,y}`) to resolve this
issue.

# Unresolved Issues

* **(Field order)**.  Should we allow `{int x, int y} <: {int y, int
    x}`.  Likewise, what about `{int x, int y, ...} <: {int x, ...}`?
    The latter is clearly problematic for runtime type tests.

* **(Covariance / Contravariance)**.  Should we support covariance /
  contravariance of function types?  The problem with allowing this is
  that it complicates runtime type testing (though perhaps does not
  make it impossible?).  Potentially, this could be relaxed at a
  further date.