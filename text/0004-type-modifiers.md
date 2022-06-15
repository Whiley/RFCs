- Feature Name: type_modifiers
- Start Date: 22/06/17
- RFC PR: https://github.com/Whiley/RFCs/pull/4

# Summary

Proposes a syntax and semantic for type modifiers which allow
restrictions to be placed on values of a type.

# Motivation

There is a clear need for the ability to express a range of simple
properties over types.  This document does not propose any such
properties but, instead, proposes a syntax and semantic for expressing
them.  Such properties are referred to as _type modifiers_.

## Examples

We now present a number of examples to illustrate some uses of such
type modifiers.  Please note that these are examples only and are not
being proposed by this RFC (though maybe proposed by later RFCs).

* The _final_ modifier would prevent variables (or parts of variables)
from being modified.  Any variable declared `final` would require an
_initialiser_.  The following illustrates:

```Whiley
final int x = 1
```

* The _atomic_ modifier would indicate a variable which can be shared
amongst one or more concurrently executing threads.  By default, this
would be sequentially consistent (following C11).  The following illustrates:

```Whiley
atomic &int x = new 1
```

* The _foreign_ modifier would indicate a type representing foreign
  data and, hence, that forms part of the _foreign function
  interface_.

```Whiley
type JsObject is foreign { ... }
```

* The _ghost_ modifier would indicate a value present purely for the
  purposes of verification.  Such values would never be compiled on
  the back-end into actual code.

```Whiley
type Purse is {
  ghost &Mint owner,
  int amount
}
```


# Technical Details

The current syntax for description of types is given as follows:

```
Type ::= UnionType
| IntersectionType
| TermType

TermType ::=
| PrimitiveType
| RecordType
| ReferenceType
| NominalType
| ArrayType
| NegationType
| FunctionType
| MethodType
| ( Type )
```

This would be extended as follows:

```
Type ::= TypeModifier* (
	UnionType
	| IntersectionType
	| TermType
	)

TermType ::= PrimitiveType
| RecordType
| ReferenceType
| NominalType
| ArrayType
| NegationType
| FunctionType
| MethodType
| ( Type )
```

The definition of `TypeModifier` would be gradually extended with
those supported type modifiers (e.g. `final`).  Note the above
definition of `Type` permits _zero or more_ type modifiers on a type.
Thus, a type of the form e.g. `final atomic int` is permitted.

### Subtyping

The interpretation of subtyping between modifiers would be determined
on a case-by-case basis as individual modifiers are introduced.  The
general assumption is that if, for two modifiers `m1` and `m2`, we
have `m1 <: m2` (i.e. that `m1` is a subtype of `m2`) then, for two
types `T1` and `T2`, we have `m1 T1 <: m2 T2` if `T1 <: T2`.

### Overloading

Whilst a semantic is given for subtyping between modified types,
another important question is whether or not function or method
overloading should be permitted.  For example, whether or not this is
valid:

```Whiley
function f(final int x) -> (int r):
   return x

function f(int x) -> (int r):
   return x
```

Whilst it may be beneficial to support such overloading in some cases
this proposal adopts a conservative approach and, as a general
assumption, does not permit overloading as a result of type modifiers.
Thus, for the purposes of overloading, the signature of a `method`,
`function` or `property` is determined by its _unmodified type_.  That
is, the type resulting after all modifiers are erased.

### Reserved Keywords.

An important question is whether or not type modifiers need to be
_reserved keywords_.  That is, tokens which cannot be used for
variable identifiers.  Whilst this can be determined on a case-by-case
basis, the simplest assumption (for now) is that they would be
reserved keywords.  This decision can be reconsidered at a later date
if necessary.

# Terminology

* **Type Modifier**.  A modifier given to a type which signals some
  additional information regarding the location holding the value of
  the given type.

* **Unmodified Type**.  A type obtained from another by erasing all
  type modifiers.  For example, the unmodified type of `final int` is
  just `int`.

# Drawbacks and Limitations

* **Reserved Keywords.** The decision to require that type modifiers
  are reserved keywords has implications on future compatibility.
  Specifically, as new modifiers are introduced, existing programs may
  no longer be syntactically correct.

# Unresolved Issues

None.
