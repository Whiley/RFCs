- Feature Name: semicolon
- Start Date: 23-07-2017
- RFC PR: 

# Summary

This proposal is to introduce a semicolon sequencing operator into
Whiley.  The purpose of this operator is to allow multiple statements
to be written on a single line.

# Motivation

Statements in Whiley must always begin on a new line, as there is no
mechanism for _sequencing_ them on the same line.  For example, in the
Java language, we can write:

```
while(i < items.length) {
  j = j + 2; i = i + 1;
}
```

The Whiley language uses indentation syntax for delineating blocks,
rather than braces (as above).  Furthermore, Whiley does not employ
semicolons for terminating statements.  Therefore, to allow multiple
statements on the same line (as above) we need some notion of
_sequencing_.  The following illustrates the proposed use of the
sequencing operator, `;`:

```
while(i < |items|) {
  j = j + 2 ; i = i + 1
}
```

The sequence operator indicates the end of one statement and the
beginning of another.

The purpose for allowing multiple statements on the same line is
simply one of convenience.  Simply put, in some cases it is useful to
do this.  One example arises when Whiley code is being presented in a
document or report.  Another situation occurs in the definition of
lambda expressions.

# Technical Details

The introduction of the sequencing operator `;` is relatively
straightforward.  Specifically, we extend the syntax for statements
with the following:

```
SequenceStmt ::= Stmt ';' Stmt
```

Observe that this definition implies that a statement must follow the
sequencing operator.  Thus, we cannot terminate a statement with ';'
and then begin a new line (as you can in Java or C).

The proposal allows sequencing between _simple statements_ only.  The
Whiley Language Specifications defines these as follows:

_"A simple statement is a statement where control always continues to
the next statement in sequence. Simple statements do not contain other
statements nested within them."_

This means there is no way, for example, to encode a `while` statement
and its body on the same line.  For example, like this in Java

```
while(i < items.length) { i = i + 1; }
```

Under this proposal, the Whiley equivalent of this requires at least
two lines (though a later RFC may further refine this).

# Terminology

- _Sequencing Operator_.  The operator `;` is the _sequencing
  operator_ used for sequencing two statements together on to the same
  line.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
