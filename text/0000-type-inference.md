- Feature Name: `type-inference`
- Start Date: `25-11-2019`
- RFC PR: (leave this empty)

# Summary

Whiley currently supports backward inference of types, including
template parameters.  This proposal aims to extend this to a more
general approach to type inference which can reduce the need to
specify template types, and introduce other improvements.

# Motivation

Currently, types in Whiley are inferred in a backwards direction.
This is best understood through the following example:

```
function id<T>(T x) -> (T y):
   return x

int i = id<int>(1) + id(2)
```

We see here an explicit type parameter is given in the first call to
`id<T>(T)` whilst, in the second, it is _inferred_ by the compiler
automatically.  In this case, the inference exploits the type given
for the argument (i.e. `2` which has type `int`).  

Backwards type inference like this works in the presence of
overloading.  For example, consider this:

```
function get<S,T>(map<S,T> map, S item) -> (T y):
   ...

function get<S,T>(map<T,S> map, T item) -> (S y):
   ...
```

When a call to `get()` is made without explicit template arguments,
the type checker must decide: firstly, which of the two functions is
being called; and, secondly, whether an appropriate binding exists.

### Forward Type Inference

An unfortunate problem occurs with the backwards algorithm as
described.  Specifically, the following fails to type check:

```
type Item<T> is { T contents }
type Node<T> is ascii::string | Item<T>

function view<T>() -> Node<T>:
    return "empty"

Node<int> n = view()
```

The problem here is that the return type for `view()` is simply
`Node<T>` for some unknown type `T`.  In fact, by looking at the type
being assigned (i.e. `Node<int>`) it is possible to determine an
appropriate type.  Using forward type inference instead of backwards
type inference is one solution to this (though not the only one).

**(Compound Types)** Forward type inference can prevent certain
amounts of unnecessary computation.  For example, consider the
following:

```
{int|null f} x = {f:0}
```

Currently, the type determined for expression `{f:0}` would be `{int
f}`.  As such, the compiler will then insert an _implicit coercion_
from `{int f}` to `{int|null f}` (which e.g. might introduce an
appropriate tag for field `f`, etc).  If, instead, the compiler was
aware that the expected type for expression `{f:0}` was `{int|null f}`
it could simply create the appropriate representation directly.

**(Primitive Types)** A similar situation to the above is the
  following:

```
int:8 f = 8
```

The type determine for `8` will be the unbound integer type `int`.
Again, this will be created at considerable expense (e.g. requiring
dynamic memory allocation) and then converted into a finite
representation.

**(Reference Types)** The following currently fails to compile:

```
&(int|null) p = new 1
```

The problem is that the type returned for `new 1` is `&int` which is
_not_ a subtype of `&(int|null)`.

**(Lambda Types)** The following currently compiles:

```
type fun_t is function(int)->(int|null)

method main():
    fun_t fn = &(int x -> x)
```

This would be implemented using a _wrapper_ for the lambda function
which, in fact, is unnecessary.

# Technical Details

The compiler already contains various components that perform forward
type inference and, hence, the primary issues are well known.  For
example:

```
(int[])|(bool[]) x = [e; n]
```

In order to type check through the expression `[e;n]` we must extract
the array element type which is then propagated through expression
`e`.  The current solution is to narrow down the target based on the
return type for the array.  However, this doesn't work with a pure
forward propagation approach and, hence, a **bidirectional** approach
is required.

**(Error Messages)** Another challenge arises with reporting error
messages.  For example, consider this (incorrect) program:

```
function id(int x) -> (int y):
   return x

function id(bool x) -> (bool y):
   return x

int x = id([1])
```

This fails to compile and reports the following error message:

```
main.whiley:7: unable to resolve name (is ambiguous)
        found function id(bool)->(bool)
        found function id(int)->(int)

int x = id([1])
        ^^
```

With an approach based on forward type inference, the question is:
_where should the error be reported?_ One possible error message might
be this:

```
main.whiley:7: expected int or bool found array

int x = id([1])
           ^^^
```

An interesting question is: _which of the two error messages is
better?_

**(Implicit Coercions)** Another issue arises with forward propagation
  and the optimal placement of coercions.  For example:

```
{int f} rec = ...
int|null x = rec.f
```

A naive implementation of forward type inference would effectively
result in this:

```
{int f} rec = ...
int|null x = (({int|null f})rec).f
```

This not at all efficient because we want to coerce the value of `f`
when it is read, rather than coercing the entire enclosing value
(which may be very expensive).

**(Overloading)**.  The presence of function or method overloading
  presents problems as well.  For example, consider this variation on
  the `id()` example from above:

```
function id(int x) -> (int y):
   return x

function id(bool x) -> (bool y):
   return x

int x = id(1)
```

The key challenge here is how we go about selecting a type to
propagate through the argument.  There are two options: `int` or
`bool` and we cannot narrow this down further until we actually visit
the argument.

## Constraint-Based Type Inference

The Whiley compiler already uses the proposed approach during the
_binding_ mechanism for type inference of function or method
invocations.  Specifically, it employs a _constraint set_ over
template and lifetime variables which is narrowed until either a
single item remains (success), no items remain (failed) or more than
one item remains (ambiguity).  

Roughly speaking, the proposed approach is based around the idea of
_constraint variables_ and _constraint sets_.  Instead of propagating
types through expressions we propagate constraint variables with lower
and/or upper bounds.  Constraints are added to the overall constraint
set which, eventually, is solved to reveal either a solution, or a
failure of some kind.

The following illustrates the simplest possible example:

```
int:8 x = 1
```

The type checker will propagate the constraint variable `c1` where
`{c1 <: int:8}` down through the assigned expression.  Upon reaching
the leaf node, the type checker adds the constraint `int:2 <: c1`.  At
this point, all constraints from the statement have been generated and
it can be solved to find the _least upper bound_ for all variables
(`c1` gives `int:8` in this case).  It then remains to update the
recorded return types / signatures for all expressions.  Observe that
this is relatively straightforward since each constraint variable
corresponds to the return type of exactly one expression.

As another example, consider the following again:

```
function id(int x) -> (int y):
   return x

function id(bool x) -> (bool y):
   return x

int x = id(1)
```

The type checker propagates the constraint variable `c1` where `{c1 <:
int}` down through the assigned expression.  At the invocation, the
type checker forks the constraint set adding `int <: c1` and `c2 <:
int` on one branch and `bool <: c1` and `c2 <: bool` on the other.  It
then propagates `c2` through the argument expression.  Again, having
generated all possible constraints from the given statement it will
attempt to solve them.

### Failures & Ambiguities

A _failure_ occurs when no solution for one or more constraint
variables can be found.  For example, consider the following:

```
int x = false
```

The final constraint set is `{ bool <: c1 <: int }`, for which no
solution exists.  Hence, a typing error is reported.  Since `c1`
corresponds to the return type of the assigned expression, the error
is reported on the expression `false`.

An _ambiguity_ arises when there is more than one solution for a given
variable.  For example, consider the following:

```
(int[])|(bool[]) xs = []
```

In this case, the final constraint set is `{ c1 <: int[] }*{ c1 <:
bool[] }` which is a forked constraint set.  The fork has arisen
because the array initialiser has forced a requirement that `c1` has
array type.  Unfortunately, at this point, we cannot eliminate either
fork and, hence, two solutions exist for `c1`.

Observe that the above is not currently detected during type checking
and, instead, relies on a subsequent check for ambiguous coercions.

### Templates

Finally, we consider the handling of template parameters.  Consider
the following:

```
type Box<T> is { T item }

function id<T>(Box<T> box) -> T:
   return box.item

int i = id({item:0})
```

The constraint set generated for the final statement is: `{c2 <: c1 <:
int, {int item} <: c3 <: Box<c2>}`.  Here, `c2` corresponds to the
return type of the invocation, whilst `c3` the type of its argument.

### Compound Accesses

Consider again this example:

```
{int f} rec = ...
int|null x = rec.f
```

The constraint set generated for the final statement is: `{c2.f <: c1 <: int|null, {int f} <: c2}`

# Drawbacks and Limitations

Unknown.

# Unresolved Issues

* **(Algorithm)** A key question is how best to solve this efficiently.
