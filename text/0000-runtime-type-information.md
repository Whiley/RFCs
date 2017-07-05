- Feature Name: runtime_type_information
- Start Date: 04/07/2017

# Summary

This RFC attempts to clarify various issues surrounding the use of
runtime type information.  In particular, whilst this RFC does not
propose any new features for Whiley, it should provide a foundation
upon which backends for Whiley can be implemented.

# Motivation

The presence of union types and runtime type testing presents several
challenges when compiling Whiley programs for different targets.
Consider the simplest example:

```
type Point is {int x, int y}
type nPoint is null|Point

function f(nPoint p) -> (int r):
   if p is Point:
      return p.x + p.y
   else:
      return 0
```

Here, the key question is how the expression `p is Point` is
implemented.  There are essentially two primary ways we might go about
this:

1. **Runtime Interrogation**.  In this situation, the runtime type
   test operator (i.e. `is`) recursively interrogates values to
   determine whether they match the given type.  This requires
   sufficient type information for values in the underlying
   compilation target.  Java is one platform which provides sufficient
   information.  Others, such as JavaScript require supplementary
   information in some cases.  Yet others, such as C, do not provide
   any such information.

2. **Tagged Union Representation**.  In this situation, every instance
   of a union type holds a finite tag which is used to distinguish
   values during type tests.  This tag is a form of _Runtime Type
   Information (RTTI)_.

For the purposes of this document, we are ignoring the first approach
above.  This is because: firstly, there are relatively few platforms
where it makes sense; secondly, it is not generally that efficient.

## Finite Types

We begin with the simple case of _finite types_.  A finite type is one
contains a finite number of _distinct types_.  For example, `int|null`
is a finite union as it contains two distinct types: `int` and `null`.
In contrast, `!null` is _infinite_ as it contains an arbitrary number
of distinct types, such as: `int`, `bool`, `{int f1}`, `{int f1, int
f2}`, etc.

As expected, a finite type can employ an integer tag for
distinguishing its members.  This is roughly similar to the concept of
a [tagged union](https://en.wikipedia.org/wiki/Tagged_union) or
[sum type](https://www.quora.com/What-is-a-sum-type).  The only real
difference here from standard approaches is the lack of explicit type
constructors.  However, we can infer tags based on context to work
around this.  For example, consider a declaration such as this:

```
type tagged is T0 | T1 | ... | Tn
```

Assuming `tagged` is finite, we can begin by allocating each case a
unique integer tag.  For simplicity, assume the tag for `T0` is `0`,
for `T1` is `1`, etc.  Then the system can automatically insert tags
in situations such as the following:

```
function f(bool flag) -> int|null:
   if flag:
      return 1
   else:
      return null
```

At each `return` statement, an implicit coercion is inserted which
converts the corresponding untagged primitive value into a tagged
union type.  Whilst this approach is relatively neat, there are
several challenges to overcome:

- **Retagging**.  Values must be retagged as they transition between
different unions.  The following illustrates:
  ```
  function f(bool flag, int|null val) -> bool|int|null:
    if flag:
      return flag
    else:
      return val
  ```
  At the statement `return val` an implicit coercion is required to
  _retag_ `val` from type `int|null` to `bool|int|null`.  Based on the
  scheme outline here this retagging can be achieved (in this case) simply
  be incrementing the tag value.

- **Overlaps**.  The approach outlined above works well when each case
  is distinct.  _But, what if they are not distinct?_ With general
  unions, as found in Whiley, this is possible.  For example, consider
  this situation:

  ```
  type Overlap is {int|null x, int y} | {int x, int|null y}

  function createOverlap(int x, int y) -> Overlap:
    return {x:x, y:y}
  ```
	
  The question is: _what tag should be used here?_ Either tag `0` or
  `1` are applicable.  A simple solution is to allocate the first
  applicable tag.  Since we decide what tag to allocate at compile
  time, this can be done using the subtype operator.
  
- **Nested Unions**.  When a union is nested within another union, we
  can choose to use a single tag or nested tags.  For example,
  consider this:

  ```
  type Nested is null | {int|null value}

  function createNested(int|null v) -> Nested:
    return {value: v}
  ```

  Since type `Nested` is equivalent to `null|{int value}|{null value}`
  we could allocate tags based on that expansion.  This means a value
  of type `Nested` contains only one (outermost) tag.  The advantage
  of this is that type testing is constant time.  The downside,
  however, is that this introduces more retagging.  For example, in
  the above, parameter `v` has an outermost tag value and must retag
  this at `return` statement.

  The alternative here is to retain nested tags.  Thus, an instance of
  `Nested` contains two tags:  the outmost tag distinguishes between
  `null` and `{int|null value}`;  the inner tag is used on field
  `value` to distinguish between `int` and `null`.  The advantage of
  this is that assigning to field `value` from a value of type
  `int|null` does not require retagging.  The downside is that type
  testing is no longer constant time but, rather, is bounded by the
  depth of tags required.  That is, for a variable `x` of type
  `Nested`, a test of the form `x is {int value}` requires checking
  both outer and inner tags.

- **Arrays**.  For a type such as `(int|null)[]` an interesting
  question is how the tags should be managed.  Following our logic
  above for nested unions, the conclusion is that the array itself is
  not tagged, but each element of the array its tagged.  This seems
  less than optimal from a performance perspective.  For example,
  testing whether it is an instance of `int[]` requires checking each
  element in the array!  However, it's interesting to note that there
  are only three distinct state that an instance of `(int|null)[]`
  could have.  Either its an instance of `int[]` (i.e. all elements
  are `int`), or an instance of `null[]` (i.e. all elements are
  `null`) or actually an instance of `(int|null)[]` (i.e. contains a
  mixture of both types).  This suggest an interesting optimisation
  where an outer tag is used.

Despite these challenges above, it does seem that tagging of finite
types is doable.

## Infinite Types

Dealing with infinite types is significantly more challenging than for
finite types.  We begin by considering the different kinds of infinite
types:

- **Any**.  The simplest kind of infinite type is one involving `any`.
  For example, `any`, `{any f}`, `any[]` are all infinite types.

- **Negations**.  The next simplest kind of infinite type is one
  involving a negation.  With the exception of `!any`, all true
  negations are infinite types.  For example, `!int`, `!{int f}`,
  `!(int[])`, are all infinite types.

- **Open Records**.  A surprising example of an infinite type is the
  open record.  For example, `{int f, ...}` contains an infinite
  number of subtypes, including `{int f}`, `{int f, int g}`, `{int f,
  int g, int h}`, etc.

- **Recursive Types**.  Another source of infinite types are recursive
  types.  In fact, every recursive type is infinite.  For example,
  type `List` defined as `null | { List next, int data}` contains an
  infinite number of subtypes, including `null`, `{null next, int
  data}`, `{ {null next, int data}, int data}`, etc.

The most important point regarding infinite types is that we cannot
realistically provide a single integer tag to distinguish their
subtypes.  Therefore, we must provide some other kind of tag to
distinguish them.  The question then is: _what does this tag look
like?_

# Technical Details

# Terminology

- **Type Tag** or **Concrete Type**.

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
