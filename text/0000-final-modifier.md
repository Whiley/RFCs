- Feature Name: Final Modifier
- Start Date: 02/06/17
- RFC PR:

# Summary

This proposes the addition of a final modifier which has a similar
meaning to that found in Java.

# Motivation

There are two primary motivations for this change: _ownership_ and
_constants_.

## Ownership

Ownership remains a thorny issue for compiler back-ends.  For example, consider the following cases:

```
function f(int[] xs) -> (int r):
    ...

function g1(int[] xs) -> (int r):
    return f(xs)

function g2(int[] xs) -> (int r):
    int r = f(xs)
    return |xs| + r
```

In this case, we do not know the contents of function `f()`.  Function `g1()` is the typical and easy case.  Ownership for the array referred to by `xs` is simply transferred to `f()` without being copied.  Function `g2()` represents the more challenging case.  In the normal course of events, the compiler is forced to copy `xs` at the point of the invocation.  This is because `xs` is still live after the invocation.  In a situation where `f()` modifies `xs`, then this is exactly the right thing to do.  But, what about the situation where `f()` does not modify `xs`?  _Then, we've made a needless copy_.

## Constants

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

# Technical Details

Definite Unassignment Analysis to complement the existing concept of
definite assignment analysis.

# Terminology

- **Definite Unassignment Analysis**.  This is the process of checking
  that a variable marked `final` is never assigned after initialisation.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
