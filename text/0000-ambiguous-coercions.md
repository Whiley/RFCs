- Feature Name: `ambiguous_coercions`
- Start Date: `13-10-17`
- RFC PR: (leave this empty)
- See also: _([RFC#0017](https://github.com/Whiley/RFCs/blob/master/text/0017-runtime-type-information.md))_

# Summary

The presence of arbitrarily complex types makes it impossible (in
general) to decide appropriate runtime tag information.  As such, this
proposal restricts the permitted implicit coercions to those which can
be statically determined.

# Motivation

The presence of true union types (i.e. [non-disjoint or untagged
unions](https://en.wikipedia.org/wiki/Union_type#Untagged_unions)) in
Whiley is a powerful and expressive feature, compared with the
alternatives (e.g. [disjoint/tagged
unions](https://en.wikipedia.org/wiki/Tagged_union), or [algebraic
types](https://en.wikipedia.org/wiki/Algebraic_data_type)).  However,
this flexibility at the source level comes at a price at the code
generation level.  Specifically, appropriate tags must be inferred
during code generation.  The following illustrates a common scenario:

```TypeScript
type msg is {int kind, int payload}|{int kind, int[] payload}

function msg(int kind, int payload) -> (msg r):
	return {kind: kind, payload: payload}
```

At the point of the return, an _implicit coercion_ is inserted to
transform a value of type `{int kind, int payload}` to a value of type
`msg`.  The latter requires a single tag bit to distinguish the two
cases.  In this case, the code generator can automatically determine
that the coercion targets the first case from the available type
information.

Whilst the above was relatively straightforward, this is not always
the case.  Consider this variation on the above:

```TypeScript
type msg is {int kind, int payload}|{int kind, int|null payload}

function msg(int kind, int payload) -> (msg r):
	return {kind: kind, payload: payload}
```

The intuition here might be that the first case is preferred since it
occupies less space, but the second is provided for general usage.
_In the above, it is unclear what the appropriate tag should be_.
This is because either case is a valid supertype of `{int kind, int
payload}`.  However, we should note that `{int kind, int payload}` is
_more precise than_ (i.e. is a subtype of) `{int kind, int|null payload}`.
These leads to the first rule of coercions:

**Rule 1:** _Most precise case always preferred._

In other words, the above program is deterministic and will always
associate the returned value with the first case of the union.

Again, whilst the above could be resolved using the available type
information, this is not always the case.  In particular, we can
easily construct examples where neither is more precise than the
other.  We refer to such situations as requiring _ambiguous
coercions_.  The following illustrates yet another variant:

```TypeScript
type msg is {int|null kind, int payload}|{int kind, int|null payload}

function msg(int kind, int payload) -> (msg r):
	return {kind: kind, payload: payload}
```

Again, either case is valid for the return value as `{int kind, int
payload}` is a subtype of both `{int|null kind, int payload}` and `{int
kind, int|null payload}`.  However, neither of the alternatives is more
precise than the other.  As such, _the code generator cannot
statically determine an appropriate type tag in this case_.  This
leads to the second rule of coercions:

**Rule 2:** _Ambiguous coercions are type errors._

In other words, the compiler is required to report an error message
for the above program, such as the following:

```
example.whiley:3: ambiguous coercion
    return {kind: kind, payload: payload}
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Whilst the presence of an ambiguous coercion is a compile-time error,
it can be easily overcome by inserting an appropriate cast for
disambiguation.  The following illustrates a corrected version of our
example:

```TypeScript
type msg is {int|null kind, int payload}|{int kind, int|null payload}

function msg(int kind, int payload) -> (msg r):
	return ({int kind, int|null payload}) {kind: kind, payload: payload}
```

At this point, the compiler now has enough information to determine
the appropriate type tag.

# Technical Details

The technical implementation of this RFC is relatively
straightforward, though there are some interesting issues to be
considered.

## Examples

There are a large number of ways in which an ambiguous coercion can
occur.  The following illustrates a few more cases to give an idea of
the scope.

**Trivially Ambiguous.** The presence of _trivially ambiguous_ union
  types should be detected and reported as an error.  For example:

```
type ambiguous is int|int
```

  In this case, there is no way in which an ambiguous type can be
  resolved (i.e. even with casting).  Such cases should be reported
  with an `ambiguous union` error.

**Nested Unions.** Many examples of ambiguous coercions (such as those
  shown above) arise from the presence of nested unions
  (e.g. `int|null`) in our example above.

**Open Records.** Whilst many examples involve nested unions, there is
  a class of examples which do not.  Essentially, all of these involve
  open records in some fashion.  A typical example would be something
  like this:

```TypeScript
type rec is {int x, ...} | {int y, ...}

function create() -> rec:
	return {x:0, y:0}
```

Again, this coercion is ambiguos as `{int x, int y}` is a subtype of
both `{int x,...}` and `{int y,...}`, but neither are subtypes of each
other.

**Intersections.** A similarly awkward situation arises with
  intersections.  The following illustrates:

```TypeScript
type from_t is (int[]|int|null)&(int|bool|null)

function convert(from_t x) -> (int|null y):
   return x
```

Here, the type `(int[]|int|null)&(int|bool|null)` simplifies to
`int|null` and, hence, the above coercion constitutes a _retagging_.

**Recursive.** Whilst these are essentially just variations on those
  examples above, it's important to take recursive types into account
  properly.  Here's one artificial example:

```
type List1 is null|{List1 next, int x, int y}
type List2 is null|{List2 next, int|bool x, int y}
type List3 is null|{List3 next, int x, int|bool y}

function f(List1 x) -> (List2|List3 y):
    return x
```

This provides the canonical example of how recursive types cause
ambiguous coercions.  As an aside, the coercion in this case is
potentially quite expensive as it must retype the entire tree.

## Algorithm

The algorithm determining whether or not a coercion is ambiguous
operates in a similar fashion as to that for resolving method/function
invocations.  Specifically, it iteratively expands nominal types until
either no more expansion is possible, or the coercion has been
resolved.

The starting point is the desire to check whether sometype `S` can
flow unambiguously into another type `T`.  A simple example
illustrating this would be:

```
function eg(S x) -> (T y):
   return x
```

We know that `T :> S` must hold (i.e. `S` is a subtype of `T`), and we
assume for now that `T <: S` does not hold (intuitively as, otherwise,
no coercion would be necessary).  We employ `<~` to denote a coercion
check, as opposed to a subtype check.  We assume that `T` and `S` are
first _simplified_ (see below for details).  Then, we have four main
cases:

* **(Untagging)** `T <~ S1 | ... | Sn`.  This case is relatively
  straightforward as its reduces to `n` independent checks `T <~ S1`,
  ..., `T <~ Sn`.

* **(Tagging)** `T1 | ... | Tn <~ S` where we assume `S` is not a
  union (i.e. as this is caught by first case above).  In this case,
  we determine the most precise case `Ti <: S`.  If no such type
  exists, then we have an ambiguous coercion.  Otherwise, we proceed
  to recursively check `Ti <~ S`.

* **(Expansion)** `T <~ S`.  Here, we assume neither `T` not `S` are
  unions and can be expanded/simplified.  Then, we _expand_
  `T` to `T'` and `S` to `S'` and check `T' <~ S'` (see below for
  details of expansion process).

* **(Decomposition)**. `T <~ S`.  Here, we assume `T` and `S` are
  compatible compound types and we recursively decompose them. For
  example, `T[] <~ S[]` reduces to `T <~ S`.  Likewise, `{T1 f, T2 g}
  <~ {S1 f, S2 g}` reduces to `T1 <~ S1` and `T2 <~ S2`.

In addition, we must ensure termination through _coinduction_.  That
is, if `T <~ S` is encountered whilst checking `T <~ S` then we assume
no ambiguos coercion.

### Simplification

Simplification is the process of exposing the _constructive_ parts of
the type by eliminating or pushing down the _non-constructive_ parts
(i.e. intersections and negations).

Simplifying a type `T` is denoted by `<T>` and there are several
cases:

* **(Primitives)**.  For example, `<bool>` and `<int>` are already
simplified.

* **(Compounds)**.  For example, `<T[]>` reduces to `<T>[]` and,
  likewise, `<{T1 f1, ... Tn fn}>` reduces to `{<T1> f1, ... <Tn>
  fn}`.

* **(Intersections)**.  An intersection involving a union
  _distributes_ over that union (e.g. `(int|null)&int` becomes
  `(int&int)|(null&int)`.  Likewise, intersections over primitives are
  evaluated (e.g. `int&int` becomes `int` whilst `int&null` becomes
  `void`).  Finally, intersections over compounds are recursively applied
  (e.g. `T[] & S[]` becomes `(T&S)[]`).

* **(Differences)**.  An difference involving a union _distributes_
  over that union (e.g. `(int|null)-int` becomes
  `(int-int)|(null-int)`.  Likewise, intersections over primitives are
  evaluated (e.g. `int-int` becomes `void` whilst `int-null` becomes
  `int`). Finally, differences over compounds are recursively applied
  (e.g. `T[] - S[]` becomes `(T-S)[]`).

* **(Unions)**.  Here, `void` elements are eliminated
  (e.g. `void|null` becomes `null`).  _Are they flattened as well?_

Since simplification does not inspect nominal types, this process is
  guaranteed to terminate.

We now illustrate this process through some examples:

* `(int|null)&int` reduces by distribution to `(int&int)|(null&int)`
  and then by elimination to `int`.

* `(null|Point)&{int x, int y}` reduces by distribution and
  elimination to `Point&{int x, int y}` which cannot be further
  simplified without expansion (see below).

* `{int|null f}&{int f}` reduces to `{(int|null)&int f}` and then by
  distribution to `{(int&int)|(null&int) f}` and, finally, by
  elimination to `{int f}`.

* `(int|null)-int` is reduced by distribution to
  `(int-int)|(null-int)` and, in turn, to `null`.

### Expansion

The expansion process takes a type and (where possible) expands it by
one level.  The expansion process takes all nominal types at the
outermost level and replaces them by their definition.  The outermost
level here includes any type reachable from the root via a type
combinator.  After expansion, we always further simplify the type by
pushing through intersections and differences to expose its top-level
structure.

For example, let's assume these type definitions:

```
type Point is {int x, int y}
type nPoint is null|Point
```

Then, consider the following ambiguous coercion check:

```
null|Point <~ nPoint
```

This cannot proceed under the other rules above since the type is not
of the appropriate form.  Instead, we apply expansion and reduce it to
the following check:

```
null|{int x, int y} <~ null|Point
```

Observe how expansion does not guarantee that all nominal types are
removed.  However, repeated expansion will eventually terminate
(i.e. since types are contractive).

## Examples

We now consider some more interesting examples to illustrate some of
the finer points.  The following does not generate a ambiguous
coercion error:

```TypeScript
type pos is (int x) where x > 0
type neg is (int x) where x < 0

function create(pos x) -> (pos|neg r):
   return x
```

The reason for this is that expansion is not applied before a tagging
operation is determined.  In contrast, this does generate an ambiguous
coercion error:

```TypeScript
type pos is (int x) where x > 0
type neg is (int x) where x < 0

function create(int x) -> (pos|neg r)
requires x > 0:
   return x
```
   
The problem here is that `int <: pos` and `int <: neg`, thereby
leading to the ambiguous coercion.

# Terminology

* **Ambiguous Coercion.** This arises when the compiler is unable to
    determine a unique target type for a given coercion.

# Drawbacks and Limitations

**Nominal Types.** As for function/method selection, the use of
nominal types should ideally be accounted for.  Specifically, consider
a situation such as the following:

```TypeScript
type msg_m1 is {any kind, int payload}
type msg_m2 is {int kind, any payload}

type msg is msg_m1|msg_m2

function create(msg_m1 x) -> (msg r):
   return x
```

In this case, it seems pretty reasonable that this coercion is not
considered ambiguous.  However, under the algorithm described above,
it will be.  This same issue currently stands for method/function
selection when resolving invocations.  Therefore, it will be corrected
in a subsequent RFC.

# Unresolved Issues

**Expansion.** The proposed expansion process is perhaps too coarse-grained?  For
example, consider this:

```TypeScript
type Point is {int x, int y}
type aPoint is Point|{int x, int y}

function f(Point p) -> (aPoint r):
   return p
```

Checking `aPoint <~ Point` requires expansion which gives `Point|{int
x, int y} <~ {int x, int y}`.  This results in an ambiguous coercion
when it seems like it didn't need to.

**Subtyping**.  The examples above tacitly assume some properties of
subtyping which are not currently true.  For example, in checking
`Point|{int x, int y} <~ Point` the assumption is that `Point` is
somehow more precise that `{int x,int y}`.  Intuitively this makes
sense, but algorithmically ... _what does it mean?_

