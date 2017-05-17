- Feature Name: operator_precedence
- Start Date: 13-05-2017
- RFC PR: (leave this empty)

# Summary

This RFC proposes the removal of operator precedence from the Whiley
language and, instead, to require that braces are used to disambiguate
expressions involving operators.  Thus, braces are required for `a +
b * c` but not for `a + b + c`.  Whilst removing operator precedence
in this manner may seem radical, it has been pioneered successfully in
the Pony language.  Furthermore, removing operator precedence makes it
significantly easier to subsequently introduce operator overloading
into the language.

# Motivation

The Whiley language uses operator precedence rules for resolving
ambiguities in parsing mixed operator expressions.  Specifically, the
rules determine which operators should bind more tightly that others.
The current set of rules are based upon those of the Java Language and
are defined in the
[Whiley Language Specification](http://whiley.org/download/WhileyLanguageSpec.pdf).

Operator precedence has historically been a thorny issue for
programming languages.  One must choose appropriate rules for the
language which, in many cases, are not at all obvious.  For example,
Java and Python use different rules for operator precedence (see
[here](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/operators.html)
and [here](https://docs.python.org/2/reference/expressions.html)).  In
addition, programmers can easily apply the rules of operator
precedence incorrectly, and there have been numerous cases of this in
practice (see
[here](http://www.spinics.net/lists/target-devel/msg09653.html) and
[here](https://patchwork.kernel.org/patch/5899871/)).

There is a desire to introduce operator overloading into the Whiley
language.  Operator precedence remains a significant impediment to
doing this (though, as illustrated in C++, it is still possible).
Removing operating precedence simply eliminates any issue related to
operator precedence for overloaded operators.

Finally, we should note that the
[Pony programming language](http://ponylang.org/) also does not have
rules for operator precedence and requires expressions to be
disambiguated with braces.  This proposal is largely based on the
approach taken in Pony.

# Technical Details

We define four operator classes as follows:

* **Infix Operators**.  This class represents operators which accept two
  ore more operands with an infix syntax (e.g. `e1 + e2`). This class
  includes the usual range of common binary operators.  More
  specifically: _infix arithmetic operators_ ( `+`,`-`,`*`,`/`,`%`),
  _arithmetic comparators_ (`==`,`!=`,`<`,`<=`,`>=`,`>`), _logical
  connectives_ (`||`,`&&`,`==>`,`<==>`), and _infix bitwise operators_
  (`|`,`&`,`^`,`<<`,`>>`).

* **Prefix Operators**.  This class represents operators which accept a
  single expression that occurs after the operator (e.g. `!e`).  More
  specifically: _prefix arithmetic operators_ (`-`), _prefix logical
  operators_ (`!`), _prefix bitwise operators_ (`~`), _prefix
  reference operators_ (`*`).

* **Postfix Operators**.  This class represents operators which accept a
  single expression that occurs before the operator (e.g. `x.f`).
  This class includes only the record and package access operator
  (`.`).

* **Mixfix Operators**.  This operator class represents operators which
accept two (or more) operands but which are non-infix operators. This
class includes the array access (`[_]`).

We now define the following two kinds of ambiguous operator expression:

* **Ambiguous Infix Expression**.  An infix expression is ambiguous if
  it involves more than one operator without disambiguating braces.
  For example, `x + y - z` is ambiguous, whilst `x + (y - z)` is not.

* **Ambiguous Prefix/Postfix Expression**.  An expression is ambiguous
  if it involves a prefix and postfix operator without disambiguating
  braces.  For example, `!x.f` is ambiguous, whilst `!(x.f)` is not.

These are the only forms of ambiguous operator expressions consider in
this proposal.  Thus, expressions involving both infix and either
prefix or postfix operators are not ambiguous.  For example, `a && !b`
is not considered ambiguous and, in such case, prefix/postfix
operators always bind more tightly than infix operators.

### Error Messages

When encountering an ambiguous operator expression, the compiler
should report an error on the left-most symbol where the ambiguity
becomes first apparent.  The following illustrates:

```
./test.whiley:5: ambiguous expression encountered (braces required)
   return x + y - z
                ^
```
Ideally, the compiler messages would evolve to make a suggestion of an
sensible disambiguation based on standard mathematical precedences.
# Terminology

* *Ambiguous-Operator Expression*.  An ambiguous-operator expression
  is one which features multiple operators at the same level
  (i.e. without braces).  For example, `a + b * c` is a mixed-operator
  expression, but neither `a + b + c` nor `a + (b * c)` are.

# Drawbacks and Limitations

* *Backwards Compatibility*.  This proposal will certainly break
  existing code.  Updating existing code is tedious but, fortunately,
  there is not much existing code at this stage!

* *Programmer Burden*.  This proposal will put more burden on
  programmers to use braces which, at first, may be frustrating.
  Ultimately, this is an unavoidable cost of this proposal.  On the
  positive side, the Pony language has reported that users quickly get
  used to this and even like it.  Furthermore, for a language (Whiley)
  which is design to eliminate errors, it seems reasonable that we
  should be strict about potentially ambiguous expressions.

Both of these limitations could be alleviated in a number of ways.  In
particular:

* **Keyword-based logical
  connectives**. [Python supports](http://stackoverflow.com/questions/16679272/priority-of-the-logical-statements-not-and-or-in-python)
  `and` and `or` as keywords for logical connectives alongside `&&`
  and `||`.  Keywords could be automatically given a different
  precedence from regular operators.  For example, `((a+(b*5))>c) &&
  (b<(3*c))` would become `(a+(b*5))>c and b<(3*c)` which is an
  improvement.

* **Precedence strictness modifier**.  We could additionally include
  some kind of file-level modifier.  For example, putting something
  like `precedence whiley.lang.Math` at the top of the file would
  provide a way to customise operator precedence.  This is nice
  because it separates precedence out of the core language.  Though,
  in reality, we would want to discourage its use.

# Unresolved Issues

* **Function Invocation**.  Should function invocation be consider an
  operator.  For example, is `!f()` an ambiguous operator expression?
