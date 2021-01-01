- Feature Name: `array-iteration`
- Start Date: `02-01-21`
- RFC PR:

# Summary

Arrays cannot be iterated directly in `for` loops or `some` / `all`
quantifiers.  This proposal aims to fix this.

# Motivation

Currently, arrays cannot be iterated directly and, instead, must be
iterated using explicit index variables.  For example:

```
function sum(int[] arr) -> (int r):
   int m = 0
   for i in 0..|arr|:
     m = m + arr[i]
   return m
```

Here, `i` is the index variable used to _indirectly iterate_ over the
array `arr`.  In some situations, it is helpful to allow _direct
iteration_ as a short form:

```
function sum(int[] arr) -> (int r):
   int m = 0
   for v in arr:
     m = m + v
   return m
```

In addition to for loops, iteration should be supported in `all` /
`some` quantifiers:

```
function max(int[] arr) -> (int m)
// Cannot compute max of empty array
requires |arr| > 0
// Result cannot be smaller than any element
ensures all { v in arr | v <= m }
// Result must match at least one element
ensures some { v in arr | v == m }:
   ...
```

This provides a fairly convenient short-hand notation which improves
overall readability.

# Technical Details

This is relatively simple to implement, and requires relatively little
implementation work.  One of the key challanges is that, for some
backends, implicit index variables need to be created.  To avoid name
clashes these can exploit the underlying _expression index_.

# Terminology

   * **Array Iteration.** This refers to the process of iterating an
       array directly, as enabled through this proposal.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.