- Generics
- 20-12-2018
- RFC PR: (leave this empty)

# Summary

This change involves adding "generics" in the style of Java Generics or C++ templates.  The change requires a syntax for declaring a type abstraction (`function foo<T>(T a, T b) -> (T ret)`), instantiating a type abstraction (`foo<int>(5, 5)`), type checking rules for generics, and evaluation rules for generics.

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

We will ignore the termination condition as it has no effect on this proposal and you can write a void-returning version that does not require it.  The `state` data type will be _whatever variables would otherwise be needed as global variables to support the animation_ and thus _are specific to the animation being run_.  I.e. there is no way to determine once-and-for-all what this state must be, it must be determined on an animation-by-animation basis by the programmer.

All is well then, we can define our state and `draw` function thus

~~~~~
type state is {int xpos, int ypos}

function draw(state in) -> (state out):
  drawCircle(xpos, ypos)
  return {xpos = in.xpos + 1, ypos = in.ypos+1}
~~~~~

And tha animation is driven with `animate(&draw, {xpos = 0, ypos=0})`

However, this program won't draw anything until the whole animation is completed.  To actually implment a real animation we must use some feature of the underlying machine on which we are running to call `draw` once, then schedule (after the frame has been drawn to the screen) the next call to `draw`.  This is simple enough in each of the back-ends that Whiley supports (Java and Javascript) but is different for each.  There is no Whiley primitives to support describing this once in Whiley and then compiling it to different back-ends.

Furthermore, a function like `animate` should be in a library somewhere regardless of implementation details.

When we attempt to define a library version of `animate` we hit an impass.  There is no type we can use for `state`.  Options we might consider, but which don't work (and why) are:
  * Pre-define the `state type`: only a limited number of animations can be supported, no extenstion is possible
  * Use a top-type so subtyping can provide the required polymorphism: Whiley currently has no top type and, furthermore, contravariance in the input type of the `draw` paramter prevents such a solution.  Relying on subtyping to allow polymorphism is not actually possible in this particular case!

What is needed here is genuine - invariant - parametric polymorphism, i.e a type variable that can be used in place of `state` and instantiated to the particular `state` the client needs.


# Technical Details

The following decisions are driven by the niche in which Whiley sits.  I.e. one would have a different answer to the questions for langauges with different goals.  Thus I note here a few characteristics of Whiley that readers will (hopefully) agree with, and which guide the decision making:
  * Whiley presents as a structurally typed imperative language with subtyping of records.
  * Whiley is really a hybrid imperative/functional language
  * Primitive types remain separate from records in Whiley for performance reasons, both at run-time (as in C et. al.) and at verifiction time (uniquely to Whiley).
  * Whiley's syntax is an amalgum of c-like and python-like.
  * Whiley strongly preferences static checks over run-time checks.

A consideration of generics must also consider if they apply at the function level only, or at the module level as well.  We address only the function level on the basis that the Whiley module system in underdeveloped at the moment.

## Syntactic Changes

Generics require syntax for the introduction of a type variable (type abstraction) and the initialistion of such variables (type application).

### Type abstraction

We propose taht all functions and methods _may_ have a set of type parameters in addition to their value parameters.  These are defined between pointy brackets and no kinds are given.  I.e. a function `foo` taking two generic parameters and then a value parameter for each will be written as

~~~~~
function foo<T,P>(T paramOne, P paramTwo) -> (T ret):
~~~~~

The justification for this syntax is:
  * it is well known from Java, C#, etc.
  * it well represents the underlying mechanism (type abstraction)
  * it fits well with the run-time behaviour proposed below.

Alternatives that were considered and the reasons they were rejected are:
  * `[T,P]` a-la Scala:  In the absence of a good reason, it is best to prefer the more common notation.
  * implicit type parameter abstraction a-la Haskell and ML:  In OO languages, such behaviour is uncommon and thus unexpected.  Personal experience has also suggested that making something implicit should only be considered where some other force is acting.  I.e. in these languages types can be inferred and thus sometimes _no_ type signature is given.
  * `template<T>` annotation a-la C++:  This notation hints better at a compile-time implemenation (such as we suggest below) but the scope of the abstraction of `T` is not clear and the now-more-common C# and Java syntax is preferrable in our opinion.

### Type application.

The syntax for type application flows fairly directly from the decision above.  A call to `foo` would require both type parameters to be instantiated with statically-known types at the call-site and then matching value parameters to be immidiately given.

~~~~~
foo<int, char>(5, 'c')
~~~~~

An alternative here is to _separately_ instantiate the types and _later_ call that instatiation normally, eg:

~~~~~
instatiate foo<int, char>

foo(5, 'c')
~~~~~

This precludes the chance to have two different instatiation in one scope, introduces an unecessary keyword, and the scope of the instatiation needs defining to complete the extension.  None of these problems exist in the other solution.

Whiley also allows aliasing function types, in which case a syntax to desribe the type of a generic function is required.  Whiley does not name (value) parameters in function types, giving only their types.  The analogue for generic paramters would be to give only the kind, but:
  * we don't consider kinds
  * the generic parameter is used later in the function signature

Thus we propose no kind is given for generic parameters, but they are named so the number of them is known and their relationship to the remainder of the type is known.

~~~~~
type GFun is function<P,Q>(P)->Q
~~~~~

NB:  I am not happy with the inconsistency here, but can't think of a better alternative.

## Semantic Changes

A _generic function definition_ such as

~~~~~
function foo<T,P>(T paramOne, P paramTwo) -> (T ret):
  return paramOne
~~~~~
 
Is a function-like entity (we will call a "pending generic") that is waiting for two type parameters.  Providing both those parameters creates a function that results from the textual substitution of the two type parameters.  I.e. providing `int` and `char` results in the creation of the function identical to

~~~~
function foo(int paramOne, char paramTwo) -> (int ret):
  return paramOne
~~~~

The scope of this pending generic is exactly as it would be for the equivalent non-generic function `foo`.

A _generic function call_ such as 

~~~~~
int v = foo<int,char>(5,'c')
~~~~~

is evaluated _at compile time_ to code equivalent to

~~~~~
function foo_int_char(int paramOne, char paramTwo) -> (int ret):
  return paramOne

int v = foo_int_char(5,'c')
~~~~~

i.e. the pending generic is instantiated to the required function immediately above the statement containing the generic function call and the statement is modified to call that function instead.  We will call this process called "generic expansion".

If there is already an identical function built from a pending generic (not user-defined) already in scope, then the pending generic may not be re-instantiated.

### Foreign functions

This style of compile-time (and macro-like) implementation of generics has some known weaknesses, in particular separate compilation is not possible for generic definitions.  Generic functions are not functions, thus they cannot be compiled.  The compiler can't know how much memory to set aside or exactly what operations to call for overloader operations.  Only the instantiations can be compiled.  C++ has the same problem and the solutions there are not particularly elegant.  However, only lanuages with either a top-type or run-time support for generics can do better and neither of these are suitable for Whiley.  Thus, we live this this problem.

More seriously though, this also precludes generic foreign functions, clearly a problem for us since that is exactly the use-case that has motivated this RFC!

I propose an extension, that I have not seen elsewhere, to support foreign generic functions.  In short, foreign generic functions are not expanded, not converted, and the type information in the generic application is used only for type checking.  This means that it is the responsibility of a foriegn function to ensure it can work on any value it may be passed.  This is not an undue burdon for the added functionality because foreign functions are effectively part of the run time and thus the writer is keenly aware of what possbile run-time types it will see.  Notice also that generics here are invariant only - i.e. we don't have examples where anything other than an equality operator is expected to be available in the body of the generic functions.  This restriction is vital to making this work and were bounded generics available in the language, only unbounded generics should be supported on foreign generic definitions.

## Compiler Changes

Type checking of generic code will need to be incorporated.  We cannot rely on the easier approach of expanding before type checking because of generic foreign functions.

For a generic function definition

~~~~~
function foo<P,Q>(P a, Q b, Q, c) -> (P ret):
~~~~~

we check the validity of a call to that function

~~~~~
foo<p,q>(aa,bb,cc)
~~~~~

as the conjunction of:
  * aa ∈ [P/p,Q/q]a
  * bb ∈ [P/p,Q/q]b
  * cc ∈ [P/p,Q/q]c

and the type of this expression is [P/p,Q/q]ret

I.e. we build a substitution for each generic parameter, apply that substitution to each value parameter type, and then perform normal type checking.

## Run-time Changes

A convention for the nameing of generic foreign functions will be required.  At the moment their types are added to their names but those can't be known for generic functions.

# Terminology

  * **Generic Method/Generic Function:** Any method or function with at least one generic parameter
  * **Generic Method Type/Generic Function Type:**  Any function or method type with at least one generic paramter
  * **Generic Arity:** The number of generic parameters required for a generic method or function.  Zero for normal methods and functions
  * **Generic Expansion:** The compile time operation that converts uses of generic functions/methods into their instatiation and use.
  * **Generic Instantiation:** The non-genric instantiation (i.e. the normal function) that results from generic expansion.
  * **Type Abstraction:** The `<G,H>` part of a function/method signature
  * **Type Application:** The `<G,H>` part of a function/method call
  * **Generic Foreign Fucntion:** Any foreign function with a generic arity greater than 0.

# Drawbacks and Limitations

As defined above, this will complicate the compiler - in particular the type checker.  However, the effect on the run-time is minimal.  The primary drawback is that this is quite a limited implementation of generics.  It supports generic foreign functions, generic collections, and operations on those collections such as map/fold.  It does not support generic code that requires any operations (besides equality or a provided function parameter) on the generic value.  It is very much a minimal implementation of parametric polymorphism via a generics mechanism.

# Unresolved Issues

Exactly what the type checking algorithm is to be and exactly what run-time changes are needed.
