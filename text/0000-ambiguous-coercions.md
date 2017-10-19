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
type msg is {int kind, int payload}|{int kind, any payload}

function msg(int kind, int payload) -> (msg r):
	return {kind: kind, payload: payload}
```

The intuition here might be that the first case is preferred since it
occupies less space, but the second is provided for general usage.
_In the above, it is unclear what the appropriate tag should be_.
This is because either case is a valid supertype of `{int kind, int
payload}`.  However, we should note that `{int kind, int payload}` is
_more precise than_ (i.e. is a subtype of) `{int kind, any payload}`.
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
type msg is {any kind, int payload}|{int kind, any payload}

function msg(int kind, int payload) -> (msg r):
	return {kind: kind, payload: payload}
```

Again, either case is valid for the return value as `{int kind, int
payload}` is a subtype of both `{any kind, int payload}` and `{int
kind, any payload}`.  However, neither of the alternatives is more
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
type msg is {any kind, int payload}|{int kind, any payload}

function msg(int kind, int payload) -> (msg r):
	return ({int kind, any payload}) {kind: kind, payload: payload}
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


**Any.** The most common kind of example (such as those discussed
  above) involves the type `any`.  Such examples require a composite
  type which has at least two or type positions.  At the time of
  writing, only records and lambdas meet this requirement.
  Nevertheless, we can still build arbitrarily complex examples by
  nesting these within other general types.  A simple variation on the
  above would be:

```TypeScript
type arr is {any x, int y}[] | {int x, any y}[]

```

This simply nests record types within an array type to generate the
necessary conditions for ambiguity.

**Open Records.**  Whilst many examples involve the `any` type, there
  is a class of examples which do not.  Essentially, all of these
  involve open records in some fashion.  A typical example would be
  something like this:

```TypeScript
type rec is {int x, ...} | {int y, ...}

function create() -> rec:
	return {x:0, y:0}
```

Again, this coercion is ambiguos as `{int x, int y}` is a subtype of
both `{int x,...}` and `{int y,...}`, but neither are subtypes of each
other.

**Negations.** A slightly more insidious approach is through the use
  of a _negation type_ to hide the necessary union.  The following
  reconsiders the above example:

```TypeScript
type rec is !(!{int x, ...} & !{int y, ...})

function create() -> rec:
	return {x:0, y:0}
```

Observe that `!(!{int x, ...} & !{int y, ...})` is semantically
equivalent to the original type `{int x, ...} | {int y, ...}`.  Thus,
this example is really identical to the above and serves only to
illustrate that algorithms for detecting ambiguous coercions cannot
just look for unions.

**Intersections.** A similarly awkward situation arises with
  intersections.  The following illustrates:

```TypeScript
type from_t is (int[]|int|null)&(int|bool|null)

function convert(from_t x) -> (int|null y):
   return x
```

Here, the type `(int[]|int|null)&(int|bool|null)` simplifies to
`int|null` and, hence, the above coercion constitutes a _retagging_.

## Algorithm

The algorithm determining whether or not a coercion is ambiguous
operates in a similar fashion as to that for resolving method/function
invocations.  We assume for now a _target type_ of the form `T1 | .. |
Tn` and a _source type_ `S` (for the expression being coerced).  Then,
under this RFC, there are two stages:

1. **Filtering**. The algorithm begins by filtering all candidates
which are not supertypes of the source type (i.e. where `Ti :> S` does
not hold).

2. **Selection**.  The algorithm selects all types `Ti` from the
remaining candidates where no `Tj` exists such that `Ti :> Tj`.

At this point, we have a list of zero or more remaning candidate
types.  For now, we assume this list is non-empty (i.e. since othewise
a type error would have occurred).  If the list has exactly one
element, then the coercion is **not** ambiguous and we can determine
the necessary tag information.  If the list has more than one element,
then we have an **ambiguous coercion**.

We now examine some of the more tricky aspects of the algorithm.

### Type Representation

The most challenging aspect of the algorithm is the question over how
exactly types are viewed.  The algorithm assumes that the target type
is a union type of the form `T1 | ... | Tn`.  But, what about types
such as `(int|null)&(int|null|bool)` or `!(!int&!null)`?  Both of
these types are really equivalent to `int|null`.

The obvious solution to the issue here is to first simplify types into
_Disjunctive Normal Form (DNF)_.  Thus, both
`(int|null)&(int|null|bool)` and `!(!int&!null)` are automatically
simplified `int|null`.

* What about nested union types?

# Terminology

* **Ambiguous Coercion.** This arises when the compiler is unable to
    determine a unique target type for a given coercion.

# Drawbacks and Limitations

**Nomihnal Types.** As for function/method selection, the use of
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

Does this make sense?  It would be treated as an ambiguous coercion
under this RFC, though perhaps not with subsequent extensions.

```TypeScript
type pos is (int x) where x > 0
type neg is (int x) where x < 0

function create(pos x) -> (pos|neg r):
   return x
```
