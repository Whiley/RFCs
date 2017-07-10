- Feature Name: final-modifier
- Start Date: 02/06/17
- RFC PR: https://github.com/Whiley/RFCs/pull/5

# Summary

This proposes the addition of a `final` modifier which has a similar
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

Ownership here refers to the internal mechanism used within a compiler
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

### Foreign Function Interface

The `final` modifier offers some benefits for the foreign function
interface.  For example, consider this simple Java class:

```
class Test {
  int f(int x) { return x; }
}
```

Without the `final` modifier, the closest type in Whiley for representing
the type `Test` would be:

```
type Test is {
  function f(int) -> (int r),
  ...
}
```

**NOTE:** An _open record_ is used to account for subclasses of
`Test`.  Also, for simplicity, we are ignoring members introduced by
`Object`, etc.

The problem with the above is that it permits assignment to the field
`f`.  That is, the following is valid Whiley:

```
export method broken(&Test t):
   t.f = &(int x -> x)
```

This is problematic if we desire the Whiley type `Test` to represent
actual instances of the Java class `Test`.  Specifically, the problem
is that actual instances of class `Test` do not support assignment to
methods.  However, using the `final` modifier we can more accurately
model class `Test` with the following Whiley type:

```
type Test is {
  final function f(int) -> (int r),
  ...
}
```

This representation of class `Test` has the desired effect of
rendering incorrect the method `broken()` above.

**NOTE:** Whilst Java does not permit assignment to methods it should
be noted that other languages, such as JavaScript, do support such
assignments.

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

### Subtyping

An important question is how subtyping is handled in the presence of
`final` modifiers.  This is relatively straightforward in that a type
`final T` is considered distinct from `T` but they are both assignment
compatible (i.e. subtypes).  Thus, this is considered valid:

```
final int x = 1
int y = x
```

Likewise, the converse is also considered valid:
```
int x = 1
final int y = x
```
(there remain some questions as to how to safely deal with memory
management here)

However, since `final T` is distinct from `T` it follows that this is
not valid:

```
&int x = new 1
&(final int) y = x
```

And, likewise, that the converse is not valid either.

**NOTE:** an interesting question is whether or not we should be
permitted to cast over `final`.  For example, whether `&(final int) y
= (&(final int)) x` should be permitted or not.  Clearly, allowing
this would break the usage of `final` in the context of a foreign
function interface (where, in that case, `final` is being used as a
_capability_).

### Definite Unassignment Analysis
   
_Definite Unassignment Analysis_ is the process of checking that a
`final` variable is not assigned.  This complements the existing concept
of definite assignment analysis.  For example, the following
fails definite unassignment analysis:

```
function f(final int x) -> (int r):
   x = x + 1
   return x
```

This should report an error roughly as follows:

```
test.whiley:2: error: final parameter x may not be assigned
	x = x + 1;
	^
```

(This is the error message given in Java for an attempt to assign to a
`final` parameter)

An important question is whether or not definite unassignment analysis
is checked in a _flow insensitive_ or _flow sensitive_ fashion.  For
example, the following is permitted under a flow sensitive analysis
but not under an insensitive one:

```
function f(bool flag) -> (int r):
   final int y
   if flag:
      y = 1
   else:
      y = 2
   return y
```

The key here is that a flow sensitive analysis can determine variable
`y` has _exactly one assignment_ regardless of which execution path is
taken and, hence, that the above is safe.  This is the approach, for
example, currently taken in Java and is considerably more flexible
than the flow insensitive approach.

**For the purposes of this proposal, definite unassignment analysis is
assumed to be a flow insensitive activity**.  This is easier to
implement that its flow sensitive counterpart.  Furthermore, we can
update this in a subsequent RFC if a more flexible approach is
considered useful.

# Terminology

- **Definite Unassignment Analysis**.  This is the process of checking
  that a variable marked `final` is never assigned after
  initialisation.

- **Flow (In)Sensitive Analysis**.  A flow sensitive analysis can take
  into consideration the context in which a statement occurs, whilst a
  flow insensitive analysis cannot.  For the purposes of this
  proposal, this means a flow sensitive definition unassignment
  analysis can take into consideration the number of other assignments
  may have occurred to a given variable prior to the current
  assignment being considered.

# Drawbacks and Limitations

- **Backwards Compatibility**.  One obvious issue is that of binary
  compatibility between different versions of an API.  Consider having
  released a `public` function or method with a `final` parameter.  If
  in a subsequence version of that function the given parameter was no
  longer marked `final`, then this could potentially lead to a hidden
  linking error.  That is, code compiled against the initial version
  would still appear to run against the later version, and yet this
  would clearly be unsafe.  One way to protect against this would be
  to include the `final` modifier in the name mangle.  Thus, its
  removal would generate a different mangle and lead to a link-time
  error.

# Unresolved Issues

The benefits of supporting `final` for reasons of ownership remain
somewhat unclear at this stage.  Here are the known issues:

- *Ownership & Cloning*.  The suggested approach to parameters would
  mean that assigning a `final` parameter to a non-`final` variable
  requires a clone.  Yet there are certainly some cases where this
  would lead to unnecessary cloning as well.

- *Ownership & Memory Management*.  In addition, there is the question
  of memory management.  That is, who is responsible for deallocating
  a `final` variable?  The suggested approach for `final` parameters
  would require responsibility to rest with the caller.

An alternative would be to introduce a specific modified
(e.g. `borrowed`) to handle the issues of ownership that are being
proposed here to be handled by `final`.  However, that might be
considered somewhat ugly as we would now have two modifiers instead of
one.
