- Feature Name: `existentials`
- Start Date: `27-08-2019`
- RFC PR: https://github.com/Whiley/RFCs/pull/54
- _Superseeded by [RFC#64](https://github.com/Whiley/RFCs/blob/master/text/0064-references.md)_

# Summary

References in the Whiley programming language do not currently support
subtyping (for good reasons).  However, subtyping is useful in
situations where it can be performed safely.  For example, in
modelling the JavaScript DOM, use of reference subtyping is essential.
This RFC proposes to support reference modifiers which enable
subtyping to be used safely and, in particular, to model foreign
objects (e.g. the JavaScript DOM).

# Motivation

An _open record_ in Whiley has an unknown number of fields.  The
following illustrates:

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
with varying payloads.  Open records behave like other value types in
Whiley and support subtyping.  For example:

```
msg m = {kind:1}
...
m = {kind:2, other:2}
```

As a consequence, they are also treated like other value types with
respect to references.  This means the following is *not* permitted:

```
&msg_i p = new {kind:1,data:2}
&msg q = p // not permitted
```

Given the current interpration of open records, this is an entirely
rational response from the type checker.  Specifically, allowing the
above would be unsound.  For example, if this code followed:

```
*q = {kind:1}
```

Since `p` and `q` are aliases, this would break the invariant that the
target of `p` had both `kind` and `data` fields (i.e. as it would
only have the former now).

**Unfortunately, subtyping of references is actually useful in many
  situations.** For example, consider this exceprt modelling the
  JavaScript DOM:

```
public type Node is {
    // Identifies what kind of node this i
    int nodeType,    
    ...
}

public type Document is {
    // Identifies what kind of node this i
    int nodeType,
    // Create raw Element
    method createElement(js_string)->(&Element),
    ...
} where nodeType == DOCUMENT_NODE
```

The problem here is that the following doesn't compile due to a lack
of subtyping:

```
&Element elem = document.createElement("div")
document.appendChild(elem)
```

Whilst we've established above the reasons why subtyping is not
supported between references, a solution to the above would be useful.
In fact, JavaScript DOM objects have some interesting properties as we
can't interact with them as normal references.

# Technical Details

The proposal adds different kinds of reference which have different
subtyping behaviours.  The following summarises:

* `&+T` describes a reference type with _covariant_ subtyping. Such types can be read fully.  For example:

```
&+{int f, int g} p = new {f:1, g:2}
&+{int f, ...} q = p
{int f} val = *q
```

Such types guarantee the referent has _at least_ the given fields (though may have more).  In other words, that `T` is an _upper bound_.  As such they cannot be assigned to protect the referent.  That is, we cannot allow `*q = {f:1}` as this would invalidate `p`.  Observer, however, that they can still be mutated.  For example:

```
&+{int f} ptr = ...
ptr->f = 1
```

* `&-T` describes a reference type with _contravariant_ subtyping.  Such types can be assigned.  For example:

```
&{int f, ...} = p
&-{int f, int g} q = p
*q = {f:1, g:1}
```

Such types guarantee the referent can accept _at most_ the given fields (though may accept less).  In other words, that `T` is a _lower bound_. As such, they cannot be read fully to protect the reader.  That is, we cannot allow `{int f, int g} = *q` as, at any given moment in time, `q` may not refer to something which has both fields!
    
* `&?T` describes a reference type whose payload is unknown but bounded by `T`.  In other words, this is an existential type which has the given set of fields.  Such types can be neither fully read nor written, though they can still be mutated.

For example, using the above we can remodel our DOM implementation as
follows:

```
type Node is &?{ int nodeType, ... }
type Element is &?{ int nodeType, string textContent }
```

This should be interpreted as saying that `Node` is a reference to an
unknown record type containing an `int nodeType` field.  Under this
interpretation, the following is valid:

```
Element elem = ...
Node node = elem
```

The above is safe because we _cannot assign to an unknown type_.  That
is, unlike before, the following is now rejected:

```
*node = new {nodeType:1}
```

This doesn't make sense under an existential intepretation because we
do not know the complete structure referred to by `node` and, hence,
cannot write sufficient information (i.e. enough fields).  Observe,
however, that the following is perfectly acceptable:

```
node->nodeType = 1
```

# Terminology

* `&T` and `&l:T` are a _read-write_ references.

* `&+T` and `&l:+T` are _read_ references.

* `&-T` and `&l:-T` are _write_ references.

* `&?T` and `&l:?T` are _unknown_ or _existential_ references.

Observe that a _read_ reference is not the same as a _read-only_
reference as it can still be mutated.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
