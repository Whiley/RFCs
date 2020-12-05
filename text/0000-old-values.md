- Feature Name: Old Heap Values
- Start Date: `05-12-2020`
- RFC PR:

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
the _tick_ operator `'` which can be used as follows:

```
method swap(&int x, &int y)
ensures *x == 'y && *y == 'x:
    int tmp = *x
    *x = *y
    *y = tmp
```

An important question is how the two operators `*p` and `'p` are
interpreted when used in a post-condition.  There are two obvious
options:

1. Old (`'p`), new (`*p`).  In this interpretation, `'p` refers to a
location in the original heap, whilst `*p` refers to a location in the
final heap.  _This following the currently accepted interpretation._

2. Old (`*p`), new (`'p`).  In this interpretation, `'p` refers to a
location in the original heap, whilst `*p` refers to a location in the
final heap.  _This flips the currently accepted interpretation._


# Terminology

# Drawbacks and Limitations

Conflict with notation for new stuff.

# Unresolved Issues

List any currently unresolved aspects of the change proposal.  These
will need to be adequately resolved before the RFC is accepted.
Identifying these issues provides a way for the author(s) of the RFC
to leverage the community in finding appropriate solutions.
