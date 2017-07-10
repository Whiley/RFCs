- Feature Name: type_precedence
- Start Date: 02/07/17

# Summary

This proposal builds on RFC#0001 by additionally removing the concept
of operator precedence for types.  

# Motivation

Types in Whiley currently require operator precedence rules for
resolving ambiguities in parsing mixed operator
expressions. Specifically, the rules determine which operators should
bind more tightly that others.  The current set of rules are defined in the
[Whiley Language Specification](http://whiley.org/download/WhileyLanguageSpec.pdf).

The following illustrates a number of mixed operator types and how
they are currently disambiguated by the language specification:

- **(Unions and Intersections)**.  The type `int|null&int` must
  currently be interpreted as `int|(null&int)` rather than
  `(int|null)&int`.  That is, intersections bind more tightly than
  unions.

- **(Negations and Unions)**.  The type `!int|null` must currently be
interpreted as `(!int)|null` rather than `!(int|null)`.  That is,
negations bind more tightly than unions.

- **(Negations and Intersections)**.  The type `!int&null` must currently be
interpreted as `(!int)&null` rather then `!(int&null)`.  That is,
negations bind more tightly than intersections.

The current rules dictated by the language specification correctly
disambiguate unions, intersections and negations.  However,
unfortunately, _they do not correctly disambiguate negation and array
types_.  The following illustrates:

- **(Negations and Arrays)**.  The type `!int[]` can currently be
interpreted as either `(!int)[]` or `!(int[])`.  Note, the compiler
currently interprets it as `!(int[])`.

Whilst we could simply update the language specification to resolve
the above ambiguities, this proposal argues that supporting operator
precedence rules for types is not a sensible approach.  That is, types
such as `int|null&bool` should not be permitted as they can easily be
misinterpreted.  Instead, this proposal follows RFC#0001 in requiring
that braces always be used to disambiguate in mixed operator types.

# Technical Details

The current syntax for types is given in the
[Whiley Language Specification](http://whiley.org/download/WhileyLanguageSpec.pdf)
as follows:

```
Type ::= UnionType
	| IntersectionType
	| TermType

UnionType ::= IntersectionType ('|' IntersectionType)*

IntersectionType ::= TermType  ('&' TermType)*

TermType ::= PrimitiveType
	| RecordType
	| ReferenceType
	| NominalType
	| ArrayType
	| NegationType
	| FunctionType
	| MethodType
	| ( Type )

ArrayType :: = TermType `[` `]`

NegationType :: = `!` TermType

...
```

This proposal updates the syntax for types as follows:

```
Type ::= UnionType
    | IntersectionType
    | NegationType
    | ArrayType
    | ReferenceType
    | TermType

UnionType ::= TermType ('|' TermType)*

IntersectionType ::= TermType  ('&' TermType)*

NegationType :: = `!` TermType

ArrayType :: = TermType `[` `]`

ReferenceType :: = `&` TermType

TermType ::= PrimitiveType
	| RecordType
	| NominalType
	| FunctionType
	| MethodType
	| ( Type )
```

# Terminology

* *Ambiguous-Operator Type*.  An ambiguous-operator type is one
  which features multiple operators at the same level (i.e. without
  braces). For example, `int|null&bool` is an ambiguous-operator type,
  but neither `int|bool|null` nor `int|(null&bool)` are.

# Drawbacks and Limitations

* **Backwards Compatibility**.  Obviously, some existing code will no
  longer compile.

# Unresolved Issues

None.
