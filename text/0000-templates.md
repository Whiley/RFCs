- Templates
- 20-12-2018
- RFC PR: (leave this empty)
- Authors: @altmattr @DavePearce

# Summary

This change involves adding [parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism) in the style of Java Generics or C++ templates.  The change requires a syntax for declaring a _template_reminiscent of C++ (`template<T> function foo(T a, T b) -> (T ret)`), and for explicitly instantiating a template (`instantiate foo<int>`), type checking rules for type parameters, and instantiation rules for templates.

# Motivation

There are myriad examples of code that is better written with generics and the interested reader is invited to explore what use programmers in other languages use to motivate their implementations.  However, there is a specific Whiley program that is motivating this proposal so we will discuss that here.

Imagine you wanted to write some form of animation loop, free from access to any global variables.  The natural implemenation of such a function will look something like

~~~~~
function animate(state -> state draw, state initialState) -> state:
  if (endCondition):
    return state
  else :
    newState = draw(state)
    return animate(draw, newState)
~~~~~

Left unspecified in the above example are;
  * the termination condition, and
  * the `state` datatype.

We will ignore the termination condition as it has no effect on this proposal and you can write a void-returning version that does not require it.  The `state` data type will be _whatever variables would otherwise be needed as global variables to support the animation_ and thus _are specific to the animation being run_.  That is, there is no way to determine once-and-for-all what this state must be --- it must be determined on an animation-by-animation basis by the programmer.

All is well then, we can define our state and `draw` function thus

~~~~~
type state is {int xpos, int ypos}

function draw(state in) -> (state out):
  drawCircle(xpos, ypos)
  return {xpos = in.xpos + 1, ypos = in.ypos+1}
~~~~~

then the animation is driven with `animate(&draw, {xpos = 0, ypos=0})`

However, this program won't draw anything until the whole animation is completed.  To actually implment a real animation we must use some features of the underlying machine on which we are running to call `draw` once, then schedule (after the frame has been drawn to the screen) the next call to `draw`.  This is simple enough in each of the back-ends that Whiley supports (Java and Javascript) but is different for each.  There are no Whiley primitives to support describing this and then compiling it to different back-ends.

Furthermore, a function like `animate` should be in a library somewhere regardless of implementation details.

When we attempt to define a library version of `animate` we hit an impass.  There is no type we can use for `state`.  Options we might consider, but which don't work (and why) are:
  * Pre-define the `state type`: only a limited number of animations can be supported, no extenstion is possible
  * Use a top-type so subtyping can provide the required polymorphism: Whiley currently has no top type and, furthermore, contravariance in the input type of the `draw` paramter prevents such a solution.  Relying on subtyping to allow polymorphism is not actually possible in this particular case!

What is needed here is genuine --- invariant --- parametric polymorphism, i.e a type variable that can be used in place of `state` and instantiated to the particular `state` the client needs.


# Technical Details

The following decisions are driven by the niche in which Whiley sits.  I.e. one would have a different answer to the questions for langauges with different goals.  Thus I note here a few characteristics of Whiley that readers will (hopefully) agree with, and which guide the decision making:
  * Whiley presents as a structurally typed imperative language with subtyping of records.
  * Whiley is really a hybrid imperative/functional language
  * Primitive types remain separate from records in Whiley for performance reasons, both at run-time (as in C et. al.) and at verifiction time (uniquely to Whiley).
  * Whiley's syntax is an amalgum of C-like and python-like.  However, care is needed to avoid collision with the existing syntax for lifetime parameters.
  * Whiley strongly preferences static checks over run-time checks.

A consideration of parametric polymorphism must also consider if they apply at the function level only, or at the module level as well.  We address only the function level on the basis that the Whiley module system in under-developed at the moment.

## Syntactic Changes

Templates require syntax for the introduction of a type variable (type abstraction) and the instantiations of such variables (template instantiation).

### Template Declaration

We propose that all properties, functions, methods and types _may_ have a set of type parameters in the form of a template declaration.  These are defined using the syntax `template <T1,..,Tn>` for arbitrarily many types `Tk`.  For example, a template type `pair` with two type parameters will be written as

~~~~~
template<S,T>
type pair is {S first, T second}
~~~~~

The justification for this syntax is:
  * it is well known from C++ and similar to that found in Java, C#, etc.
  * it well represents the underlying mechanism (type abstraction)
  * it fits well with the run-time behaviour proposed below.

Alternatives that were considered and the reasons they were rejected are:
  * `[T,P]` a-la Scala:  In the absence of a good reason, it is best to prefer the more common notation.
  * implicit type parameter abstraction a-la Haskell and ML:  In OO languages, such behaviour is uncommon and thus unexpected.  Personal experience has also suggested that making something implicit should only be considered where some other force is acting.  I.e. in these languages types can be inferred and thus sometimes _no_ type signature is given.

Finally, we note that the `template<T>` notation hints at a compile-time implemenation (such as we suggest below).

### Template Instantiation

A key challenge in the implementation of templates is to physically instantiate them.  In C++, this was an issue for a long time but explicit instantiation was introduced in C++11 (see [here](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.3.0/com.ibm.zos.v2r3.cbclx01/explicit_instantiation.htm)).  A _template function declaration_ such as

~~~~~
template<T,P>
function foo(T paramOne, P paramTwo) -> (T ret):
  return paramOne
~~~~~
 
Is a function-like entity parameterised by two (unknown) types.   In order to use our template for `foo` we must explicitly instantiate it as follows.
 
~~~~~
instantiate foo<int, char>
~~~~~

Providing both those parameters creates a function that results from the textual substitution of the two type parameters.  I.e. providing `int` and `char` results in the _creation_ (in the enclosing compilation unit) of the function identical to

~~~~
function foo(int paramOne, char paramTwo) -> (int ret):
   return paramOne
~~~~

At this point, we can invoke `foo()` as though it were a locally defined declaration.  A key challenge is that `template` declarations are type checked.  Therefore, for example, the following template declarations will be rejected:

```
template<T>
function f(T x) -> (int r):
   return x
```
```
template<T>
function f(T x) -> (T r):
   return x+1
```
These are both rejected because it is not certain that `T` will be instantiated as an `int` type.  This is _despite the fact that they would be sensible if instantiated as such_.

## Implementation Details

There are three key technical issues with implementing this proposal:

  * **(Type Checking)** Template declarations will need to be type checked which, in turn, will necessitate the introduction of something akin to `SemanticType.Variable`.  Type checking itself is relatively straightforward at this stage because a type parameter is only a subtype of another type parameter _with the same name_.   The `FlowTypeChecker` will need to be updated to include an environment for type parameters.  This could probably just be a set of declared type parameters even.  Alternatively, the parser could be updated to do this for us.
  * **(Name Resolution)** Resolving names will be somewhat more challenging.  Identifying that a given name corresponds to an instantiation should be easy.  However, at this time, declaration uses are linked directly to their declarations (e.g. `Decl.Function`) but, in this case, there would be no corresponding declaration.
  * **(Termination)** As with C++ templates, the issue of infinite loops during instantiation remains a problem.  This proposal makes no suggestions on resolving this and notes only that it is a compile-time issue and, hence, less important.

Various other stages would need to be updated (e.g. `DefiniteAssignmentAnalysis`, `VerificationConditionGenerator`, etc) --- most of which should be straightforward.

## Extensions

Finally, extensions which could be implemented after the fact include:

- **(Conditional Compilation)**.  Following C++, having some notion of a `const expr` would allow for compile-time specialisation.  This moves things towards a macro system, rather than just a template system.  I think that would be good (see below for example).  See [here](https://github.com/Whiley/WhileyCompiler/issues/726) for some thoughts on this.
- **(Type Bounds)**.  I think it would be potentially good to support type bounds as well.  That way, we can say that a generic type has some structure (e.g. its a record with at least these fields).  However, care is needed here.

A good use for conditional compilation would be to support the automatic generation of methods for converting between different types.  For example, consider a library for manipulating JSON.  In Whiley we might write this:

```
type payload is { int x, int[] fields }
```

What we want is to be able to automatically construct a `toJSON` function for this type.  To do this, the JSON library can provide a template `toJSON` which includes a bound forcing the type parameter to be a record, and a loop that iterates the fields of the type parameter generating the necessary conversions for each field.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
