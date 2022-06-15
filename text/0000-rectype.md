- Feature Name: `rectype`
- Start Date: `15-06-22`
- RFC PR: (leave this empty)

# Summary

Recursive types can be defined in Whiley using the `type` keyword.
Whilst this is syntactically neat, it does not give any indication to
the compiler has to how best to *implement* them.  This RFC proposes a
requirement that recursive types are defined using `rectype` instead
of `type`.

# Motivation

Recursive types are easily defined in Whiley using `type` as follows:

```Whiley
type List is null|{int data, List next}
```

This type is _recursive_ because it is defined in terms of itself.  As
such, any implementation on a low-level platform will have to
implement the type using a reference.  We say that the recursive type
must be _broken_ into its flat and reference components.  To
understand this better, lets imagine one "low-level" encoding of the
above:

```
typedef List = null | &{int data, List next}
```

Here, a variable of type `List` is a union of `null` and a reference.
On a 64-bit architecture, for example, this requires `65` bits to
encode (ignoring trickery with word aligned pointers). An alternative
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

Whilst this example is contrived, it illustrates the issue: where do
we break the recursive type `Message`?  There are various places it
could be broken.  For example, `Payload` could be implemented as a
reference; or, `Payload` could be implemented as a flat structure.
Leaving the compiler to make this decision is somewhat risky.  For
example, passing recursive types across FFI boundaries would require a
deep understanding of the algorithm.  Furthermore, it might be subject
to change as the compiler evolves.  Therefore, instead, _requiring_
programmer direction on the breakpoint is the intent of this proposal.

# Technical Details

This should provide a detailed discussion regarding the technical
aspects of the proposal.  This should clearly identify what exactly is
being changed (e.g. what the new syntax is, what the new feature does,
etc).  This should also detail what parts of the compiler need to be
changed in order for the change to be implemented.

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
