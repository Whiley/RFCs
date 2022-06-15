- Feature Name: `recinit`
- Start Date: `24-07-2019`
- RFC PR: https://github.com/Whiley/RFCs/pull/52

# Summary

Various programming languages (e.g. Rust and JavaScript) have adopted
simplified record initialiser syntax.  Since this has also been
identified as an issue in Whiley, it makes sense to follow suit.

# Motivation

The following illustrates a common pattern for initialising a record:

```Whiley
type Point is {int x, int y}

function Point(int x, int y):
   return {x:x, y:y}
```

This pattern occurs a lot, and is rather ugly.  JavaScript ES6
introduced simplified object literal syntax to handle this and,
likewise, Rust supports simplified syntax as well.

# Technical Details

When initialising a record, a variable can be used on its own to
initialise a field of the same name.  For example, the above can be
rewritten as:

```Whiley
function Point(int x, int y):
   return {x, y}
```

Observe that we can mix and match old-style initialisers with new
style initialisers.  For example, we can write this:

```Whiley
function Point(int y):
   return {x:0, y}
```

Finally, it is a syntax error to inialise a field more than once.
Thus, the following should not compile:

```Whiley
function Point(int x, int y):
   return {x:x, x, y}
```

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
