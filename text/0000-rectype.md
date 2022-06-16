- Feature Name: `rectype`
- Start Date: `15-06-22`
- RFC PR: (leave this empty)

# Summary

Recursive types can be defined in Whiley using the `type` keyword.
This offers syntactic simplicity but offers no hints as to how they
should be *implemented*.  This RFC proposes a requirement that
recursive types be defined using `rectype` instead of `type`.

# Motivation

Recursive types are currently defined in Whiley using `type` as follows:

```Whiley
type List is null|{int data, List next}
```

This type is _recursive_ because it is defined in terms of itself.
Any "low level" translation will need use a reference somewhere to
implement this.  We say that the recursive type must be _broken_ into
its flat and reference components.  To understand this better, imagine
an encoding of the above in an imaginary "low-level" language:

```
typedef List = null | &{int data, List next}
```

A variable of type `List` is a union of `null` and a reference.  On a
64-bit architecture, for example, this requires `65` bits to encode
(ignoring trickery with word aligned pointers). An alternative
encoding is this:

```
typedef List = &(null | {int data, List next})
```

In this case, a variable of type `List` is just a reference and,
hence, fits into `64` bits on a 64-bit architecture.

The point here is not that one representation is better than another.
Only that there are options here and the final decision is currently
left to the compiler.  We can take this a step further as follows:

```Whiley
type Message is {int header}|{int header, Payload data}
type Payload is {bytes[] bytes, null|Message sub}
```

Whilst this example is contrived, it illustrates the issue: _where do
we break the recursive type here?_  There are various places it could
be broken.  For example, `Payload` could be implemented as a
reference; or, `Payload` could be implemented as a flat structure.
Leaving the compiler to make this decision is somewhat risky.  For
example, passing recursive types across FFI boundaries would require a
deep understanding of the decision process used by the compiler.
Furthermore, it might be subject to change as the compiler evolves.
Therefore, instead, _requiring_ programmer direction on the breakpoint
is the intent of this proposal.

# Technical Details

This document proposes that every recursive type should be
_explicitly_ broken by the programmer using a `rectype` definition.
In particular, failure to do this should generate a compile-time
error.  Intuitively the intention is that, at a low level, a `rectype`
is implemented as reference.  

## Overview

Consider again our `LinkedList` example.  Under this proposal, our
previous definition of `LinkedList` would generate a compile-time
error.  However, we can update it to use `rectype` as follows:

```Whiley
rectype List is null|{int data, List next}
```

This now compiles, and explicitly demarks `LinkedList` as a recursive
type.  Furthermore, the expected translation of this into our
low-level language would be:

```
typedef List = &(null | {int data, List next})
```

The key is that there is _exactly one expected translation_.  If we
wanted to get something close to the other translation of `LinkedList`
shown above, we would have to write the code differently like this:

```Whiley
rectype Link is {int data, List next}
type List is null|Link
```

In this case, `Link` is implemented as a reference whilst `List` is
implemented inline.

## Nesting

Whilst it is a requirement that any recursive type is broken at least
once, it is permitted to have multiple breaks.  For example, the
following illustrates:

```Whiley
rectype Link is {int data, List next}
rectype List is null|Link
```

In this case, `List` has two breaks and now corresponds with this
low-level representation:

```
typedef Link = &{int data, List next}
typedef List = &(null | Link)
```

Here we see that both `List` and `Link` are translated as references.
Whilst this is unlikely to be an efficient implementation, the point
is that the programmer has control.  The reason for allowing this is
that we imagine situations where it is desirable or even necessary.
For example, a `Stmt` type which is itself recursive (e.g. through
compound statements such as `WhileStmt`) and contains potentially
recursive expressions (of which some, such as lambdas, might contain
statements).

## Non-recursive Recursive Types

This proposal does not require that a `rectype` is _actually_
recursive.  For example, the following should compile without error:

```Whiley
rectype Point is {int x, int y}
```

The key, however, is that this _would still be translated as a
reference_.  The reasoning here is that there is no need to prohibit
non-recursive `rectype`s (and potentially they might be useful in some
situations).

# Terminology

   * **Recursive Type**.  A _recursive type_ is one declared with `rectype`.
   
   * **Flat Type**.  A _flat (or inline) type_ is one declared with `type`. 

# Drawbacks and Limitations

N/A

# Unresolved Issues

   * **Checking**.  Its not immediately clear how best to implement
     the compiler check that recursive types are broken using
     `rectype`.  This requires some kind of depth-first search to
     identify cycles in the type graph.
