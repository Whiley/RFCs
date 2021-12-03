- Feature Name: `property_syntax`
- Start Date: `19-10-21`
- RFC PR: https://github.com/Whiley/RFCs/pull/102

# Summary

Properties provide an important mechanism for supporting verification
as, within the verifier, they are treated as _interpreted functions_.
Currently, the syntax for properties is limited to predicates which is
unnecessarily restrictive.  This RFC will take the first step towards
generalising properties accordingly.  Firstly, properties will now be
able to return arbitrary values; secondly, properties for reasoning
about heap relations (e.g. using `old(e)` syntax) will be given
special status as _variants`.

# Motivation

Currently, the syntax for _properties_ is pretty awful.  Here's an
example from the standard library:

```
// Check if two arrays equal for a given subrange
public property equals<T>(T[] lhs, T[] rhs, int start, int end)
// Arrays must be big enough to hold subrange
where |lhs| >= end && |rhs| >= end
// All items in subrange match
where all { i in start..end | lhs[i] == rhs[i] }
```

The essential point of properties (as it stands) are that they receive
special status within the theorem prover.  Specifically, properties
can be _inlined_ during the verification process. This provides
significant additional expressivity over functions which are treated
as _uninterpreted_ during verification.  In some sense, properties are
like macros.

The most obvious way to improve the syntax of properties is to allow
an expression body.  Combined with e.g. conditional expressions or
general terms, this makes them quite powerful.  A key issue arises
around what restrictions (if any) should be placed on the body of a
`property`.  Currently, properties can be recursive but, obviously,
cannot contain loops.  The suggested approach is to restrict their
bodies to a subset of statements, initially including only `return`
and `if`.  This means they cannot contain variable declarations, or
loops, etc.  An example is the following:

```Whiley
property sum(int x, int y) -> int:
   x + y
```

Obviously, this is pretty restrictive and this RFC anticipates
generalising this further in the future.  An example illustrating
control-flow is the following:

```Whiley
property sum(int[] arr, int i) -> int:
    if i >= |arr|:
        0
    else:
        arr[i] + sum(arr,i+1)
```

Henceforth, properties will employ single state reasoning --- meaning
they cannot use the `old(e)` expression form (which requires two-state
reasoning).  Instead, this RFC introduces `variant` syntax for
two-state reasoning as follows:

```Whiley
type List is null|Node
type Node is &{ int data, List next }

variant unchanged(List l)
where (l is Node) ==> (l->data == old(l->data))
where (l is Node) ==> unchanged(l->next)

method m(List l)
ensures unchanged(l):
    skip
```

Here we see a `variant` being used to specify that a given `method`
does not change the structure of a `List`.

# Terminology

Properties and variants are _interpreted_ whereas functions are
uninterpreted.  That means, when verifying a call to a function, its
implementation details remain hidden and can change arbitrarily
without affected whether or not the caller verifies.  This allows
functions which have non-trivial implementations that are subject to
change (e.g. in `std::array`).
