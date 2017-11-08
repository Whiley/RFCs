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

## Algorithm

The algorithm determining whether or not a coercion is ambiguous
operates in a similar fashion as to that for resolving method/function
invocations.  We assume for now a _target type_ of the form `T1 | .. |
Tn` and a _source type_ `S1 | .. | Sm` (for the expression being
coerced).  Then, under this RFC, for each `Sj` there are two stages:

1. **Filtering**. The algorithm begins by filtering all candidates
which are not supertypes of the source type (i.e. where `Ti :> Sj` does
not hold for some `i`).

2. **Selection**.  The algorithm selects all types `Ti` from the
remaining candidates where no `Tj` exists such that `Ti :> Tj`.

At this point, we have a list of zero or more remaning candidate types
for each `Sj`.  For now, we assume this list is non-empty (i.e. since
othewise a type error would have occurred).  If the list has exactly
one element, then the coercion is **not** ambiguous and we can
determine the necessary tag information.  If the list has more than
one element, then we have an **ambiguous coercion**.

We now examine some of the more tricky aspects of the algorithm.

### Type Representation

The most challenging aspect of the algorithm is the question over how
exactly types are viewed.  The algorithm assumes that the target type
is a union type of the form `T1 | ... | Tn`.  But, what about types
such as `(int|null)&(int|null|bool)`?  Both of these types are really
equivalent to `int|null`.

**DNF.** One solution is to first simplify types into _Disjunctive
Normal Form (DNF)_.  Thus, `(int|null)&(int|null|bool)` is
automatically simplified `int|null`.  This still raises some
questions (e.g. how to deal with recursive types).

**Flattening.** The use of DNF seems to help, but it raises questions
  about the flattening of nested unions.  For example, should
  `(int|null)|bool` be simplified to `int|null|bool`?  The usual
  simplification rules would suggest it should.  But, this has
  implications.  Consider this example:

```TypeScript
type msg is { int|null value }

function f(int|null x) -> msg:
   return {value: x}
```

There are two ways to interpret this.  At first, it might seem as
though no coercion is necessary here.  Specifically, `x` flows into
the field `value` directly.  But, under the usual rules of
simplification, `{int|null value}` would be expanded to `{int
value}|{null value}`.  In this, we would then be introducing a type
tag at the return statement.  These alternatives are discussed in
[RFC#0017](https://github.com/Whiley/RFCs/blob/master/text/0017-runtime-type-information.md)
and one approach is to have a customised set of simplification rules.
But, it's not clear how these would work.  For example, how do we
simplify `({int|null f}[])|(({int f}|{null f})[])`?  Or, perhaps more
importantly, something like `({int|null f}[])&!(({int f}|{null f})[])`?

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

Does this make sense?  It would be treated as an ambiguous coercion
under this RFC, though perhaps not with subsequent extensions.

```TypeScript
type pos is (int x) where x > 0
type neg is (int x) where x < 0

function create(pos x) -> (pos|neg r):
   return x
```
