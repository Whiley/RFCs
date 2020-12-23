- Feature Name: `old-heap-values`
- Start Date: `05-12-2020`
- RFC PR: https://github.com/Whiley/RFCs/pull/84

# Summary

Reasoning about the state of the object heap before and after a method
invocation requires specific syntax, which this proposal offers.

# Motivation

Reasoning about side-effects on the object heap requires a mechanism
for describing its state _before_ and _after_ a method invocation.  In
particular, a mechanism within the postcondition of a method for
describing the state of the heap on entry.  To begin, consider a
simple method which can be specified:

```
method put(&int p, int x)
ensures *p == x:
   *p = x
```

Currently, the use of `*p` in the postcondition refers to the state of
the heap _after the method has completed_.  This allows us to say
something about the side effects of the method.  Now, consider the
slightly more complex `swap()` method below:

```
method swap(&int x, &int y):
    int tmp = *x
    *x = *y
    *y = tmp
```

Unfortunately, there is no mechanism in Whiley for specifying this
method.  Specifically, there is no way to related the state of the
heap before the method is called with that after it is call.  This is
necessary for this method as its meaning only makes sense in terms of
the state beforehand.

Other similar languages, such as
[JML](https://en.wikipedia.org/wiki/Java_Modeling_Language),
[Spec#](https://en.wikipedia.org/wiki/Spec_Sharp) and
[Dafny](https://en.wikipedia.org/wiki/Dafny), all provide a mechanism
for refering to the _old_ state of the heap.  For example, in JML we
have `\old(e)` and in Dafny we have `old(e)` which, in both cases,
_evaluates to the value expression `e` had on entry to the current
method_.

# Technical Details

The key technical contribution of this proposal is the introduction of
the _tick_ operator `'*e` for an arbitrary expression `e`, illustrated
as follows:

```
method swap(&int x, &int y)
ensures *x == '*y && *y == '*x:
    int tmp = *x
    *x = *y
    *y = tmp
```

An important question is how the two expressions `*p` and `'*p` are
interpreted when used in a post-condition.  There are two obvious
options:

1. Old (`'*p`), new (`*p`).  In this interpretation, `'p` refers to a
location in the original heap, whilst `*p` refers to a location in the
final heap.  _This follows the currently accepted interpretation._

2. Old (`*p`), new (`'*p`).  In this interpretation, `*p` refers to a
location in the original heap, whilst `'*p` refers to a location in the
final heap.  _This flips the currently accepted interpretation._

Interpretation (1) above is perhaps the most consistent with other
tools which employ `old(e)` to capture the pre-state.  In contrast,
interpretation (2) is perhaps slightly easier to implement as the
meaning of `*p` is not context dependent (i.e. always translates to
the initial heap).  _This proposal adopts interpretation (1)_.

## Examples

The following provides a more complex example to illustrate:

```
type Option<T> is (T|null item)
type map<T> is Option<T>[]

property equalsExcept<T>(T[] lhs, T[] rhs, int i)
where |lhs| == |rhs| && all { k in 0..|lhs| | i == k || lhs[k] == rhs[k] }

property inserted<T>(Option<T>[] before, Option<T>[] after, int i, T item)
where all { k in 0..|before| | (i == k) ==> (before[k] == null && after[k] == item) }

/**
 * Insert an item into a heap-allocated map
 */
method insert<T>(&map<T> m, T item)
// Require some space in the map
requires some { i in 0..|*m| | (*m)[i] == null }
// Item has been inserted somewhere
ensures some { i in 0..|*m| | equalsExcept('*m,*m,i) && inserted('*m,*m,i,item) }:
    ...
```

Anothe interesting example is the following (which could never verify):

```
method broken(int x) -> (&int q)
requires x >= 0
ensures '*q >= 0:
   //
   return new x
```

The above could never verify because it attempts to refer to a
location in the original heap which did not exist!

# Terminology

   * *(Old vs New Heap)*.  This proposal introduces a notion of the
      heap _before_ and _after_ a given operation has executed.

# Drawbacks and Limitations

   * Other considerations are to allow for naming return values
     (e.g. if `st` is a parameter, then we can have `st'` as a return
     value to signal a connection).  Aligning this syntax would
     potentially be helpful.

# Unresolved Issues

Eventually, the notation can be generalised to `'e`' for use with
variants, etc.
