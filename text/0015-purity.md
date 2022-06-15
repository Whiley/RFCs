- Feature Name: `functional_purity`
- Start Date: `14-09-2017`
- RFC PR: https://github.com/Whiley/RFCs/pull/15

# Summary

The question of what exactly *functional purity* means in Whiley is
currently vague.  This proposal aims to clear that up.

(**NOTE:** this relates to [RFC#0008](https://github.com/Whiley/RFCs/blob/master/text/0008-global-variables.md))

# Motivation

The concept of a *pure* or *referentially-transparent function* is
relatively well understood.  For example, [Wikipedia](https://en.wikipedia.org/wiki/Pure_function) lists two
requirements:

> 1) The function always evaluates the same result value given the same
> argument value(s). The function result value cannot depend on any
> hidden information or state that may change while program
> execution proceeds or between different executions of the program,
> nor can it depend on any external input from I/O devices
> (usually—see below).
>
> 2) Evaluation of the result does not cause any semantically observable
> side effect or output, such as mutation of mutable objects or output
> to I/O devices (usually—see below).

As such, it may be surprising that there are some questions as how
this should be applied in Whiley.  There are two specific contexts we
must account for:

- **Function Context**.  This corresponds to all statements and
  expressions contained within a `function`.  This is directly
  comparable to the above statements regarding pure functions.

- **Functional Context**.  This corresponds to an expression in a
  context which should be functionally pure.  For example, all
  expressions within a precondition or postcondition should be
  functionally pure (i.e. regardless of whether they are for a
  `function` or `method`).  Likewise, presumably, for loop invariants
  and the conditions of `assert` or `assume` statements.  Finally, a
  `property` is not a function context but, rather, a functional
  context.

Taking this into consideration, there are now several issues to
consider:

**Can an expression in a function or functional context refer to a
variable of --- or containing --- reference type?**  For example, is
this allowed:

```Whiley
function cmp(&int p, &int q) -> bool:
   return p == q
```

This does not appear to violate any of the principles set out above.
Furthermore, in a functional context, it is clear that we should
support this:

```Whiley
method swap(&int p, &int q)
requires p != q:
   ...
```

Without the ability to refer to variables of reference type within a
specification element we would be completely unable to reason about
them.

**Can an expression in a function or functional context dereference a
  variable of reference type?**   This is more subtle, as the
  following illustrates:

```Whiley
function f(&int p) -> (int r):
    return *p
```


This clearly violates the first principle above but not the second
(i.e. since it does not have *side effects*).  One can attempt to
argue that this doesn't violate the principles of referential
transparency.  That is, if we consider the *wider state* referred to
by `p` then we can argue that, given the same state, the same result
will be returned (**note:** such state is often referred to as the
*frame*).

Prohibiting dereferencing in a function does not necessarily pose a
problem.  However, as before, in a functional context we might prefer
something different:

```Whiley
type Link is &{ LinkedList next, int data }
type LinkedList is null | Link

// Specify that p is contained in q
property contains(Link p, Link q)
where p == q || (q->next != null && contains(p,q->next))
```

Such a `property` is extremely useful for reasoning about linked data
structures.  This could be used, for example, to specify that a
`LinkedList` is indeed acyclic (i.e. head not contained in rest, etc).
Such properties clearly make sense in a setting where heap structures
can be manipulated.

**Can an expression in a function or functional context invoke a
method?**  This is perhaps more clear cut.  Consider the following:

```Whiley
function f(&int p, &int q) -> (int r):
  swap(p,q)
  return 0
```

It's pretty clear that allowing a pure `function` to invoke a `method`
violates the above principles and, furthermore, breaks down the
distinction between them.  *But, what about for functional contexts?*  

```Whiley
method m(&int p, &int q)
requires swap(p,q):
   ...
```

This does not make sense as the `requires` clause has a _side
effect_.  Thus, for example, whether or not this clause is evaluated as
runtime has a semantically observable effect.

# Technical Details

We now clarify the rules proposed by this RFC:

1. No expression or statement in a function context can dereference a
variable, invoke a `method`, allocate objects via `new` or access a
static variable.

2. No expression in a functional context can invoke a `method` or
   allocate objects via `new`.

Observe that, at the time of writing, functional context's are always
expressions.  Should this be relaxed, then rule (2) should be extended
accordingly.  The rough intentions of these rules are:

- That a functional context cannot have side-effects.

- That a `function` cannot access the heap.

At this point, it remains simply to clarify what exactly a functional
context is.

## Functional Contexts

A functional context is an expression placed in one of the following:

1. A `requires` or `ensures` clause.

2. A `where` clause, including for type invariants, loop invariants
   and properties.

3. The condition for an `assert` or `assume` statement.  

These, collectively, are referred to as _specification elements_.
Should additional specification elements be added to the language
(e.g. `varies`), then these would need to be included above.

Finally, _constant expressions_ are also considered functional
contexts.  Constant expressions are used, for example, for static
variable initialisers and have additional restrictions.

# Terminology

- **Function Context**.  The scope of a `function` which includes all
  specification elements, statements and expressions.

- **Functional Context**.  The scope of a specification element or
constant expression and includes all expressions contained therein.

- **Specification Element**.  A `requires`, `ensures`, `where` clause
  or the condition of an `assert` or `assume` statement.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
