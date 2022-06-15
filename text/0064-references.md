- Feature Name: `references`
- Start Date: `25-11-2019`
- RFC PR: https://github.com/Whiley/RFCs/pull/64
- Tracking Issue: https://github.com/Whiley/WhileyCompiler/issues/985
- See also: [#0054](https://github.com/Whiley/RFCs/blob/master/text/0054-existentials.md)

# Summary

The current semantics for references in Whiley are difficult (or
impossible) to represent as native references, leading to an impedence
mismstach.  This is undesirable from an interop perspetive.

# Motivation

References in Whiley are currently implemented as shared variables or
"boxes".  This is a simple and easily understood semantic where the
type `&T` is interpreted as a _reference to a variable of type `T`_.
The following illustrates:

```Whiley
&(int[]) p = ...
// Keep old value
int[] old = *p
// Store new value
*p = [1,2,3]
```

The key here is that we can read and write _complete_ values of type `T`.
This is both powerful, and problematic.  For the above example, the
machine represention after the final assignment would look something
like this:

```
+---+    +---+    +---+---+---+---+
| *----->| *----->| 3 | 1 | 2 | 3 |
+---+    +---+    +---+---+---+---+
  p                      
```

Here, we can see that `p` is implemented as a "pointer-to-a-pointer"
which is necessary to allow an array of arbitrary length to be
assigned.  **Unfortunately**, this means that the type `&(int[])` does
not correspond to an integer array in, for example, Java or
JavaScript.  We note also the issues raised in
[RFC0054](https://github.com/Whiley/RFCs/blob/master/text/0054-existentials.md)
and that _this RFC superceeds RFC#0054._ More specifically, this RFC
removes the unknown reference type `&?T`.

Finally, we note the difference between the representation of a type
`&int` and `&(int[])`.  For example, consider the following:

```Whiley
&int p = ...
// Keep old value
int old = *p
// Store new value
*p = 123
```

The representation for `p` is much simpler in this case and, in fact,
does correspond with a simple reference to an `int` variable.  The
following illustrates:


```
+---+    +-----+
| *----->| 123 +
+---+    +-----+
  p
```

Thus, we can easily represent a type `&int` as an underlying reference
type (e.g. `int*` in C).

# Technical Details

The proposal is fairly straightforward.  A variable of type `&T`
should correspond to a single reference to a representation of type
`T`.  For example, our type `&(int[])` would be represented as
follows:


```
+---+    +---+---+---+---+
| *----->| 3 | 1 | 2 | 3 |
+---+    +---+---+---+---+
  r                      
```

This means that `&(int[])` can correspond directly to an integer array
in JavaScript or Java.  Likewise, a type `&{int x}` can correspond
directly to a JavaScript object, etc.

In order to implement this proposal, we must make further restrictions
on variables of reference type.  For example, the following is no
longer permitted:

```Whiley
&(int[]) p = ...
*p = [1,2,3]
```

It should be clear that this cannot be permitted because the chunk of
memory to which `p` refers may not be sufficiently large to hold the
value `[1,2,3]`.  Furthermore we note that, whilst the following could
be supported, it will not be permitted at this time (though future
RFCs may consider supporting it):

```Whiley
int[] arr = *p
```

The key distinction made in this RFC is that a variable of type `&T`
may be fully dereferenced (read or write) only when `T` is a
statically-sized type (recall
[RFC#0003](https://github.com/Whiley/RFCs/blob/master/text/0003-statically-sized.md)).
In all other cases, only partial dereferencing is permitted
(e.g. `arr[i]` or `x.f`, etc).

The following are all permitted under this RFC:

```Whiley
&int p = ...
int old = *p
*p = 123
```

```Whiley
&{int x} p = ...
{int x} q = *p
p.x = 123
*p = {x:456}
```

```Whiley
&(int[]) p = ...
assert |p| > 0
int old = p[0]
p[0] = 123
```

```Whiley
&{int x, ...} p = ...
int old = p.x
p.x = 123
```

The syntax for accessing arrays and records defaults to the standard
and, furthermore, field dereference syntax (e.g. `p->f`) is no longer
supported.

### Boxes

This RFC proposes the introduction of a standard library type,
`std::box::Box` defined as follows:

```Whiley
public type Box<T> is &{ T contents }
```

This is similar (in some ways) to the type `std::boxed::Box` in Rust
([see here](https://doc.rust-lang.org/std/boxed/struct.Box.html)).
Using this type, we can rewrite our original example as follows:

```Whiley
Box<int[]> b = ...
// Read old valud
int[] old = b.contents
// Store new value
b.contents = [1,2,3]
```

The memory layout here is identical to that for the first example as
well.  Thus, it is clear that this proposal does not reduce the
expressivity of the language.

Finally, we note that references to template types do not support any
form of dereferencing.  For example, the following is no longer
permitted under this RFC:

```Whiley
method get<T>(&T ref) -> T:
   return *ref
```

The issue here is that, at compile time, there is no means to
determine whether `T` is a statically sized type or not.  In the
future, the introduction of type bounds could alleviate this.

# Terminology

* _Box._ This references to a variable of type `Box<T>` for some
    type `T`.

* _Partial Dereference._ Accessing a variable of reference type
  through the array or record access operators.

# Drawbacks and Limitations

**(Array Reads)**  The following code could, in principle, be supported
with the current proposal:

```Whiley
&(int[]) p = ...
// Read current value
int[] arr = *p
```

This is possible because there is enough information to construct an
appropriate value for the assignment.  This could therefore be
supported at a later date should it be deemed useful.  We note,
however, that this would not apply to open records and, hence, the
following could never be supported:

```Whiley
&{int x, ...} p = ...
// Read current value
{int x, ...} rec = *p
```

The issue is that there is not enough information to determine how
many fields there are, and what types they have.  Whilst this
information could be embedded in an open record, this would add
considerable cost to all uses of open records.

**(Array Writes)** Finally, we note the following could (in princple)
  be supported under certain conditions:

```Whiley
&(int[]) p = ...
// Write some value
*p = [1,2,3]
```

This code would generate a proof obligation that `|p| >= 3`.  In other
words, the assignment would be permitted _provided there was enough
space for it be safe_.

