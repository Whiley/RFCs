- Feature Name: `return_variables`
- Start Date: (put in today's date, `DD-MM-YYYY`)
- RFC PR: (leave this empty)

# Summary

For some time now, there has been a lack of clarity regarding how
return variables should operator in Whiley.  This RFC attempts to
clarify this once and for all.

# Motivation

Whiley has long supported the notion of _return variables_ or just
_returns_ for short.  The following illustrates:

```
function f(int x) -> (int y):
   ...
```

Here, `x` is a declared parameter and `y` a declared return.
Similarly, for some time, it has been the case that return variables
cannot be redeclared.  For example, this fails to compile:

```
function f(int x) -> (int y):
   int y = x
   return y
```

The reported error message is:

```
./test.whiley:2: name already declared
    int y = x
        ^
```

Although Whiley originally allowed you to redeclare return variables,
it was felt that this was potentially confusing --- especially when
return variables were used in specifications.

The question then is: _should we be allowed to use return variables?_
For example, at the current time, the following program does compile:

```
function f(int x) -> (int y):
   y = x
   return y
```

This is not unreasonable and we can argue it may offer some potential
benefits.  For example, we can imagine that all space for variable `y`
is allocated in the caller.  Thus, we can avoid reallocating space in
some case by reusing this preallocated space.  This would be
particularly important in the case of fixed-size arrays
(e.g. `int[8]`) on constrained environments, such as an embedded
system.

The next question is: _what does it mean to assign to a return
variable?_ We should note that there are various languages that
support this behaviour, including
[Pascal](https://en.wikibooks.org/wiki/Pascal_Programming/Syntax_and_functions)
and Dafny.  In such languages, assigning to return values _is the only
way to return a value from a function or method_.  As such, it seems
that assigning to a return value should be a substitute for returning
a value.  In otherwords, this should compile:

```
function f(int x) -> (int y):
   y = x
```

Here, `y = x` is semantically equivalent to `return x`.  At the
current time, this program does not compile however.

We note another variation on this for reference:

```
function greaterTen(int x)->(bool r):
    if (x > 10):
        r = true
        return
    r = false
```

Again, this should compile and work as if the `return` was `return
true`.  Noting, of course, there seems little advantage to using it in
this case.

# Technical Details

This could be achieved with relative ease.  Definite assignment can be
simply updated to ensure that all declared returns are defined at the
point of a `return` statement.  The verification condition generator
would also need to be updated accordingly.

# Terminology

None.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
