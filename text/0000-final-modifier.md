- Feature Name: Final Modifier
- Start Date: 02/06/17
- RFC PR:

# Summary

This proposes the addition of a final modifier which has a similar
meaning to that found in Java.

# Motivation

The primary motivations for this change are: 
_constants_, _ownership_ and _foreign code_.

### Constants

Currently, constants in Whiley are defined as follows:

```
constant JANUARY is 1
```

However, this syntax is rather clumsy.  Instead, the proposal will be
made to remove `constant` declarations and simply replace them with global
variables.  For example, this would allow us to write:

```
int JANUARY = 1
```

This is not a strict replacement for the original `constant`
declaration because it represents _mutable state_.  However, through
this proposal we could write the following:

```
final int JANUARY = 1
```

This is now a proper replacement for `constant` declarations.

### Ownership

Ownership here refers to the internal mechanism used within a compile
back-end for determining when to clone data.  The following
illustrates a minimal example:

```
function f(int[] xs) -> (int r):
   return |xs|

function g(int[] xs) -> (int r):
   ...
   int x = f(xs)
   ...
   return x
```

At the point of the function invocation, the compiler must determine
whether or not to clone the array `xs`.  In doing this, it cannot
examine the body of the function being invoked (because this requires
interprocedural analysis which is prohibitively expensive).  Rather,
the compiler can only use the function's _signature_ (i.e. name,
parameter types and `requires` / `ensures` clauses).

In deciding whether to clone `xs` above, the compiler can consider the
remainder of the function body.  There are two cases: either `xs` is
used after the inovcation; or, it is not.  If `xs` is not used after
the invocation then it does not need to be cloned.  However, if `xs`
is used after the invocation then the compiler has no choice and must
clone `xs`.  This is unfortunate in this case as the clone was
unnecessary (i.e. because `xs` is not assigned within the body of
`f()`).

To address the above issue, this RFC proposes the introduction of the
`final` modifier.  This would allow `f()` above to be rewritten as
follows:

```
function f(final int[] xs) -> (int r):
   return |xs|
```

The intention here is that the `final` modifier signals to any
invocation sites that `xs` array will not be modified and, hence, they
may pass in a _borrowed_ reference to it.

**NOTES:** I don't know whether _partial declarations_ make sense
  (i.e. for fields and primitive types)

### Foreign Function Interface

# Technical Details

The intention is that the `final` modifier can be applied to any
`lvalue` location.  Specifically, an `lvalue` identifies a variable
which occupies a specific location in memory.  The modifier is written
at the declaration site of the given `lvalue` location.  This includes
the following:

- **Local Variable Declarations**.  The following illustrates such a
usage:

```
function f(int x) -> (int r):
   final int y = x + 1
   return y
```

- **Parameter Declarations**.  The following illustrates such a
usage:

```
function f(final int x) -> (int r):
   return x+1
```

- **Type Declarations**.  The following illustrates such a
usage:

```
type fint is (final int x)
```

- **Field Declarations**.  The following illustrates such a
usage:

```
type Point is {
  final int x,
  final int y
}
```

- **Reference Declarations**.   The following illustrates such a
usage:

```
function f(&(final int) p) -> (int r):
   return *p
```
   
### Definite Unassignment Analysis
Definite Unassignment Analysis to complement the existing concept of
definite assignment analysis.

# Terminology

- **Definite Unassignment Analysis**.  This is the process of checking
  that a variable marked `final` is never assigned after initialisation.

# Drawbacks and Limitations

None.

# Unresolved Issues

- *Ownership*.  The benefits of supporting `final` for reasons of
  ownership remain somewhat unclear at this stage.  For example, the
  suggested approach would mean that when assigning a `final` variable
  to a non-`final` requires a clone.  Yet there are certainly cases
  where this would lead to unnecessary cloning as well.
