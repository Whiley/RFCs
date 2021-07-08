- Feature Name: `final_parameters`
- Start Date: `17-10-2017`
- RFC PR: (leave this empty)
- See Also: [RFC#0005](https://github.com/Whiley/RFCs/blob/master/text/0005-final-modifier.md)

# Summary

Parameters in Whiley are currently treated in the same fashion as
local variables, meaning they can be assigned within the body of a
function or method.  Unfortunately, this results in some potential
confusion when it comes to interpreting `ensures` clauses.  To prevent
such confusion, this proposal is to require that parameters are
`final` by default.

# Motivation

Parameters in Whiley are mutable and can be assigned within the body
of a function or method.  Unfortunately, this causes confusion over
the meaning of a parameter used in an `ensures` clause.  The following
illustrates:

```
function abs(int x) -> (int r)
ensures r >= 0
ensures (r == x) || (r == -x):
  //
  if x < 0:
     x = -x
  return x
```

The issue here is that the `ensures` clause refers to the value of `x`
as it was on entry to the function, rather than its value at the end.
This is potentially confusing (e.g. for students) since `r==x` would
appear to always hold at the `return` statement.

We can make an interesting generalisation here which further
illustrates the problem.  Specifically this code:

```
function f(int x) -> (int r)
requires P
ensures Q:
    BODY
    return E
```

where `E` is some expression, might (in some sense) be considered
equivalent to this:

```
function f(int x) -> (int r):
    assume P
    BODY
    assert Q[E/r]
    return E
```

But, this equivalence does not hold if `x` is allowed to change within
the body of the function.

## Solution

The proposed solution is fairly straightforward.  We simply treat
parameters as `final` by default, thereby preventing assignment to
them within the body of a function or method.  Thus, our `abs()`
method above would fail compilation with an error such as the
following:

```
abs.whiley:6: assignment to parameter not permitted
    x = -x
    ^^^^^^
```

We note that the Dafny language also does not permit assignment to
parameters, and it may be this is the reason why.

The implementation of this proposal is relatively straightforward and
employs an algorithm for _definite unassignment_ (see below).

## Implications

There are a number of interesting effects from this proposal (aside
from the potential for widespread code breakage):

**Heap Versioning Syntax.** An important question is the syntax for
  expressing the _before_ and _after_ effects of a method on the heap.
  For example,
  [WyC#750](https://github.com/Whiley/WhileyCompiler/issues/750) uses
  this example to illustrate:

```
method swap(&int x, &int y)
ensures *x == *'y && *y == *'x:
    int tmp = *x
    *x = *y
    *y = tmp
```

Here, `*'` was intended to be read as: _the state of the heap at the
point of completion_. Thus, `*'x` is here denoting the value of the
location referred to by `x` _after the method completes_.

This proposal suggests an alternative interpretation of `*'` would be
sensible.  Specifically, following the above interpretation and our
equivalence from before, we end up with:


```
method swap(&int x, &int y):
    int tmp = *x
    *x = *y
    *y = tmp
    assert (*x == *'y) && (*y == *'x)
```

Here, it makes much more sense for `*x` to refer to the current value,
and `*'x` to refer to the original value.  This would then be
comparable to the `\old()` syntax used in JML and the `old()` notation
used in Dafny.

**Cloning.** As discussed in
  [RFC#0005](https://github.com/Whiley/RFCs/blob/master/text/0005-final-modifier.md)
  there are some potential benefits from `final` parameters in the
  context of value semantics.  The minimal example is this:

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
whether or not to clone the array `xs`. In doing this, the compiler
can only use the function's signature (i.e. name, parameter types and
requires / ensures clauses).  In deciding whether to clone `xs` above,
the compiler can consider the remainder of the function body. There
are two cases: either `xs` is used after the inovcation; or, it is
not. If `xs` is not used after the invocation then it does not need to
be cloned. However, if `xs` is used after the invocation then the
compiler has no choice and must clone `xs`. This would be unfortunate
in this case as the clone was unnecessary (i.e. because `xs` is not
assigned within the body of `f()`).  Note, however, that under this
proposal `xs` would be a `final` variable and, hence, the compiler
would know that it could not be assigned within the body of the
invoked function.

**Shadow variables.** When generating code with the intention of
  performing runtime assertion checking, the compiler currently must
  insert so-called _shadow variables_ to hold the values of parameters
  on entry to the function/method.  This is necessary to that they can
  be restored before evaluating the postcondition at the point of a
  `return`.  Under this proposal, shadow variables become unnecessary
  because parameters cannot be modified.

# Technical Details

Definite unassignment is the process of checking that a variable is
not assigned again within the body of a function or method.  This is
typically implemented as a dataflow analysis, and was perhaps first
introduced in
[Java](https://docs.oracle.com/javase/specs/jls/se7/html/jls-16.html).

The implementation of this proposal employs a simplified version of
the _definite unassignment_ algorithm described in
[RFC#0005](https://github.com/Whiley/RFCs/blob/master/text/0005-final-modifier.md).
It is a simplification because we do not need to consider variables
(i.e. parameters) which are not initialised.

# Terminology

* **Definite Unassignment Analysis**. This is the process of checking
  that a variable marked final is never assigned after initialisation.

# Drawbacks and Limitations

The drawbacks are relatively obvious.  All existing code may suffer
breakage as a result of this proposal.  This includes code in
benchmark suites, test suites, published papers, student assignments,
the Whiley Language Specification, the Whiley Standard Library.  In
short, it is potentially a significant cost to update all of this
existing code.

# Unresolved Issues

None.
