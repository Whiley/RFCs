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

## Finite Type

We begin with the simple case of _finite types_.  A finite type is one
contains a finite number of _distinct types_.  For example, `int|null`
is a finite union as it contains two distinct types: `int` and `null`.
In contrast, `!null` is not finite as it contains an arbitrary number
of distinct types, such as: `int`, `bool`, `{int f1}`, `{int f1, int
f2}`, etc.

## Infinite Unions

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
