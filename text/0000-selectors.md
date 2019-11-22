- Feature Name: `type_selectors`
- Start Date: `22-11-2019`
- RFC PR: (leave this empty)

# Summary

This is an informational RFC which clarifies the notion of a _type
selector_.  The intention is that this will provide the foundation for
future RFCs related to subtyping and flow typing.

# Motivation

A type selector `s` refines a given type `T` to yield a _proper
subtype_ `S`.  For example, consider this assignment:

```
null|js_string x = "hello"
```

Let us assume the type of `"hello"` is `ascii::string` (i.e. we're
ignoring forward type inference).  We wish to refine the type of `x`
based on this assignment whilst retaining the fundamental
characteristics of its declared type.  In this case, let's suppose
that `js_string` is intended to map directly to native strings in
JavaScript.  This means that, under-the-hood, the code generated for
`x[i]` differs depending on whether `x` is `ascii::string:` or
`js_string`.

In the above we want `x != null` to be known from the initialiser.
But, what is the type of `x`?  If we simply used the type of the
initialiser itself, then it would be `ascii::string` (which would
defeat the purpose of declaring it as a native string).  Thus, we want
the type of `x` to be `js_string`.  We say that the declared type of
`x` is _refined_ by its assignment to `js_string`.

The purpose of a type selector is to capture this notion of
refinement.  Specifically, it _selects_ those parts of a given type
which are known to be active (i.e. `js_string`) whilst ignoring other
parts (i.e. `null`).  The challenge is, based on the declared type of
`x` and the type of the assigned expression, to generate an
appropriate selector.

# Technical Details

A type selector is an n-ary tree terminated by either `_` (bottom) or
`*` (top).  For example, `((*,_),*)` is a valid selector.  The
intuition is that `_` ignores the corresponding component, whilst `*`
selects it.  The notation `T o s` describes the _application_ of a
selector `s` to a type `T` resulting in a refined type.  The following
illustrates a bunch of examples:

   * `int|null o (*,_)` gives `int`.
   * `int|null|bool o (*,_,*)` gives `int|bool`.
   * `(int|null)[] o ((*,_))` gives `int[]`.
   * `{int f, int|null g} o (*,(*,_))` gives `{int f, int g}`.

Thus, we see that a type selector matches the _shape_ of the type
being selected from.  We note that, in some cases, it may be desirable
for selectors to retain tag information.  For example, we might say
that `int|null|bool o (*,_,*)` gives `int|void|bool` instead of
`int|bool` to indicate that the tag for `bool` is `2` not `1`.

Finally, we note there are currently three recognised places in the
Whiley language where refinenment occurs:

   * **Variable Declarations.** For example, `T1 x = e` where `e` has
       type `T2` leads to the refinement `T1 o s` for variable `x`.

   * **Assignments.** For example, `x = e` or `x.f = e` as above.

   * **Runtime Type Tests.** For example, `x is T2` where `x` has type
       `T1` also leads to the refinement `T1 o s` for variable `x`.

### Subtype Witnesses

A key observation is that selectors alone are insufficient for
demonstrating that one type is a subtype of another.  In other words,
they cannot act (by themselves) as a _witness_.  The intuition here is
that `T1 :> T2` iff some witness `w` exists such that `T1 o w` gives
`T2`.  In some cases, a selector is enough byitseld (e.g. for
`int|null :> int`).  However, it's not sufficient in other cases, such
as the following:

```
int|null x = ...
null|int y = x
```

The only valid selector we can generate here is `*`, but it follows
that `null|int o *` does not give `int|null`.  In fact, we need an
additional concept of a _morphism_ to given an appropriate subtype
witness.

### Error Messages

Type selectors can (potentially) be used for improved error messages.
For example, consider this:

```
{int x, int y, int z} p = { x:0, y:1, z:false }
```

An appropriate error message might indicate `{..., int z}` was
expected, but that `{..., bool z}` was given.  Here, selectors are
used to simply ignore parts which are not important.

# Terminology

  * **(Type Selector)**.  A type selector is used to extract a given
      substructure from a type.
  
  * **(Proper Subtype)**.  Let `T1` be a _proper subtype_ of `T2`.
      Then, there exists some selector `s` such that `T2 o s` gives
      `T1`.
  
  * **(Subtype Witness)**.  Let `T1` be a _subtype_ of `T2`.  Then,
      there exists some witness `w` such that `T2 o w` gives `T1`.

# Drawbacks and Limitations

None.

# Unresolved Issues

A key challenge is the handling of recursive types.  For example,
consider the following arrangement:

```
type List<T> is null | { T data, List<T> next }

List<int> x = ...
List<int|null> y = x
```

At this time, there is no possible selector `s` such that
`List<int|null> o s` gives `List<int>`.  This is because we need an
_infinite_ selector capturing the invariant that all instances of
`data` have the `int` component selected.  Our current definition of
selectors does not permit this because it only describes selectors of
_finite_ length.