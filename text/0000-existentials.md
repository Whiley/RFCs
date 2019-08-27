- Feature Name: `existentials`
- Start Date: `27-08-2019`
- RFC PR: (leave this empty)

# Summary

The Whiley programming language currently supports the concept of an
"open record".  However, open records are essentially unusable in
their current state.  This RFC proposes to change the manner in which
open records are interpreted, thereby giving them an important role
within the language.

# Motivation

An open record is simply a record which has an unknown number of
fields.  The following illustrates their current usage and
interpretation:

```
type msg is { int kind, ... }
type msg_i is {int kind, int data} where kind == 0
type msg_b is {int kind, bool data} where kind == 1

function getPayload(msg m) -> (int|bool|null r):
    if m is msg_i:
        return m.data
    else if m is msg_b:
        return m.data
    else:
        return null
```

This defines an arbitrary concept, `msg`, for representing messages
with varying payloads.  A key feature of the above is that the runtime
type test operator is used to examine `m` to decide what it actually
is.  Since `m` represents a potentially infinite number of message
types, the runtime implementation must provide meta-information
concerning (at a minimum) the number of fields and their names in
order to enable runtime type testing.

An interesting feature of open records is that they behave like other
value types in Whiley.  For example, we can assign arbitrarily to
them:

```
msg m = {kind:1}
...
m = {kind:2, other:2}
```

As a consequence of this, they are also treated like other value types
with respect to references.  This means the following is *not*
currently permitted:

```
&msg_i p = new {kind:1,data:2}
&msg q = p // not permitted
```

Given the current interpration of open records, this is an entirely
rational response from the type checker.  Specifically, allowing the
above would be unsound.  For example, suppose this code followed:

```
*q = {kind:1}
```

Since `p` and `q` are aliases, this would break the invariant that the
target of `p` had both `kind` and `data` fields (i.e. as it would
only have the former now).

**Following the above interpretation, open records do indeed make
  sense**.  However, they are not currently used much.  The intended
  use case is to enable polymorphism.  For example, the current
  implementation of `std::io::Reader` is:

```
public type Reader is {
    method read(uint) -> byte[],
    method has_more() -> bool,
    method close(),
    method available() -> uint,
    ...
}
```

Here, the intention is that future implementations of `Reader` may
contain additional (hidden) fields which constitute their
implementation state.  **However, the current implementation of Whiley
provides no mechanism enabling this**.  This is because we cannot
currently bind functions or methods to their enclosing record.

To understand this further, consider the following `Channel` interface:

```
type Channel<T> is {
     method read()->(null|byte),
     method write(byte)->bool,
     ...
} 
```

Then, suppose a (hypothetical) implementation of this using an
in-memory array:

```
type ArrayChannel<T> is {
   int[] data,
   int size,
   method read()->(null|byte)
   method write(byte b)->bool
}
```

There are a lot of problems here: firstly, _how do we instantiate
`ArrayChannel<T>`?  secondly, the return types for `read()` and
`write()` should include the updated `ArrayChannel`, etc.  Whilst
there are possible solutions here, they themselves involve some form
of existential type!  Furthermore, they are not necessarily prohibited
by the proposed treatement of open records (though this depends on
some things).

# Technical Details

The proposal is to change the interpretation of open records to have
an _existential_ semantics (i.e. similar to Java wildcards), rather
than being like other value types.  For example, consider this type:

```
type Node is &{ int nodeType, ... }
type Element is &{ int NodeType, string text }
```

This should as saying that `Node` is a reference to an unknown record
type containing an `int nodeType` field.  Under this interpretation,
the following is valid:

```
Element elem = ...
Node node = elem
```

The above is safe because we _cannot assign to an unknow type_.  That
is, unlike before, the following is now rejected:

```
*node = new {nodeType:1}
```

This doesn't make sense under an existential intepretation because do
not know the complete structure referred to by `node` and, hence,
cannot write sufficient information.  An important consequence is that
it no longer makese sense to instantiate a variable of open record
type.  For example, the following is no longer permitted:

```
function id(msg m) -> (msg r):
    return m
```

Whilst this `function` previously compiled, it would no longer compile
under the existential interpretation.  This is because we could not,
for example, invoke this function (i.e. as it requires knowledge of
the unknown type).  However, this is less of a concern as we can
employ templates to implement this method more easily at this time
(and, if type bounds are supported in the future, many other similar
methods).



# Terminology

In changing the language, it is important to develop a suitable
"vernacular" for talking about the new aspects of the language.  For
example, if a new piece of syntax is proposed, then a standard way of
referring to this should be proposed.  Likewise, if a new phase of the
compiler is introduced, then suitable terminology for discussing the
various aspects of this should be developed.

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
