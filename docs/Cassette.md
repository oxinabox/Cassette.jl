# What is Cassette?

Cassette is a Julia package that provides a just-in-time (JIT) IR pass injection mechanism,
or **overdubbing mechanism**, that can be used to interleave external code transformations
with Julia's normal JIT-compilation cycle. As part of this overdubbing mechanism, Cassette
provides a **contextual execution framework** for the Julia language via the implementation
of **contextual dispatch** and **contextual metadata propagation**. This framework enables
users to define new "contexts" that overlay normal Julia methods that can then be dispatched
on using Julia's existing operator-overloading syntax. Using its overdubbing mechanism,
Cassette can equip context-unaware Julia code with context-specific behaviors, enabling
granular method interception and metadata propagation within "black-box" user code.

Downstream applications for Cassette include automatic differentiation, interval constraint
programming, dynamic code analysis (e.g. profiling, `rr`-style debugging, etc.), JIT
transpilation to GPU backends, automatic parallelization/rescheduling, memoization,
high-level automated memory management and code fuzzing.

Outside of Julia, the LLVM ecosystem has long benefitted from modular compiler extensions in
the form of the LLVM Pass Framework. In a similar vein, we hope that Cassette's overdubbing
mechanism can eventually be used to jump-start an ecosystem of Julia compiler "plugins" -
modular extensions that implement new compiler optimizations and features external to the
compiler itself. In fact, there is already [previous work in this
regime](https://github.com/IntelLabs/ParallelAccelerator.jl) that could drastically benefit
from a formal, standardized approach to compiler extension.

# Background: Julia's Compiler

To understand how Cassette works, one must first have at least a cursory knowledge of
where Cassette fits into Julia's run-compile cycle, as well as Julia's `@generated`
function feature. An explanation of the former is given in the following section, while an
explanation of the latter is given in [Julia's official `@generated` function
documentation](https://docs.julialang.org/en/latest/manual/metaprogramming/#Generated-functions-1).

## Julia's Run-Compile Cycle

A diagram of Julia's run-compile cycle is shown below. While this diagram is quite
incomplete<sup id="f1-anchor">[1](#f1)</sup>, it is sufficient for the purposes of
understanding Cassette.

![Compilation Cycle](compile-cycle.png)

Here, the colored boxes are phases of the run-compile cycle, the arrows denote execution
flow, and the gray boxes are the input/output of each phase. The red boxes are phases that
Cassette interacts with directly. Before we examine these interactions, here's an outline
of the process described by the diagram, starting at "Function Call":

1. A function call is the application of a callable Julia object (the function) on other
Julia objects (the arguments). The function call is resolved to its corresponding method
body by dispatching on the call's type signature. For example, `f(1.0, "a", [1 + im])` has
the type signature `Tuple{typeof(f),Float64,String,Vector{Complex{Int}}}`.

2. Dispatch occurs via a lookup performed on Julia's internal method table, using the
call's type signature as a kind of key. This method lookup can yield one of two results.
If the method body for this function call has already been compiled in the past, and
the cached native code from that compilation is still valid (a determination made via
Julia's "world-age" mechanism), then the rest of the compile cycle for this function
call is skipped and the cached native code is executed (i.e., we skip to step 6).
Otherwise, the method lookup yields an untyped Julia AST, represented in Julia's internal
intermediate representation (IR).

3. Assuming that there is no valid cached native code for the function call, Julia's
compiler performs type inference on the method body AST, leveraging the function call's
type signature as a starting point.

4. Various optimization passes - most notably, inlining and `@generated` function
expansion - are interleaved with Julia's type inference process, hence the cycle
formed between "Type Inference" and "Optimizations/Inlining".

5. The type inferred and optimized IR for the method body is passed to the compiler's
code generator, where the IR is converted into LLVM bitcode, which is in turn converted
to executable native code.

6. The native code is executed, potentially returning runtime values that are then fed
to another function call, at which point the cycle repeats.

<b id="f1">[1]</b> To be considered complete, this diagram would need to also have a
"parse time" section, which would include Julia AST construction and macro expansion.
[↩](#f1-anchor)

## Where Cassette Fits In The Run-Compile Cycle

As denoted by the red boxes in the run-compile diagram, Cassette interacts with the
"Function Call" and "Optimizations/Inlining" phases. Cassette interacts with the  "Function
Call" phase by wrapping target functions in a callable `Overdub` wrapper type,  whose call
definition is a special `@generated` function. This `@generated` function  exploits the
generator expansion that occurs during the "Optimization/Inlining" phase as an injection
site for Cassette's overdubbing mechanism. In short, `@generated` call definition for
`Overdub` performs a method lookup in order to retrieve the original method's code, then
runs context-specific transformation passes on that code before returning it from the
generator.

In the case of Cassette's contextual execution framework, this overdubbing mechanism
facilitates a JIT-rewrite of the target method's code in order to propagate the context
type and any associated metadata to downstream function calls.

## Changes to Julia's Compiler

To facilitate the development of Cassette, several changes to the compiler have been
made, and several more are planned. Below is a list of relevant GitHub issues and pull
requests tracking the development of these changes:

- [JuliaLang/julia#22440](https://github.com/JuliaLang/julia/pull/22440): Allow `CodeInfo` objects to be returned directly from `@generated` functions
- [JuliaLang/julia#22938](https://github.com/JuliaLang/julia/pull/22938): Add a `CodeInfo` validator to Base
- [JuliaLang/julia#22979](https://github.com/JuliaLang/julia/pull/22979): Enable generator expansion for code reflection methods
- [JuliaLang/julia#22877](https://github.com/JuliaLang/julia/pull/22877): Improve performance of certain varargs functions
- [jrevels/Cassette#5](https://github.com/jrevels/Cassette/issues/5): Optimization of "pass-through" varargs functions
- [jrevels/Cassette#6](https://github.com/jrevels/Cassette/issues/6): World-age validation for certain kinds of `@generated` functions
- [jrevels/Cassette#7](https://github.com/jrevels/Cassette/issues/7): Generated `CodeInfo` inlining should occur in Base rather than in Cassette
- [jrevels/Cassette#9](https://github.com/jrevels/Cassette/issues/9): `getfield` projection overhead

# Cassette's Overdubbing Mechanism and Contextual Dispatch

As stated earlier, Cassette's overdubbing mechanism works by using a wrapper type whose
call definition is a `@generated` function that injects context-specific code
transformations into the compiler's "Optimization/Inlining" phase. In this section,
the overdubbing mechanism will be examined in more detail.

## A Simple Overdubbing Mechanism

Below is a simplified, mocked-out of version of Cassette's overdubbing mechanism that
only supports transformation pass injection:

```julia
# returns the world age within the current calling context
get_world_age() = ccall(:jl_get_tls_world_age, UInt, ())

# This callable struct will wrap the target function and every
# downstream function called by the target function in order to
# redirect dispatch based on the context.
#
# Note the inclusion of the current world age as a type parameter.
# This is necessary to force re-expansion/re-compilation of the call
# definition (the `@generated` function below) when the world age
# updates (for example, when a new method is added to `pass` below).
struct Overdub{F,C,w}
    func::F
    context::C
    world::Val{w}
end

Overdub(func, context) = Overdub(func, context, Val(get_world_age()))

# Given a context type `C`, returns a transformation pass of the form
# `f(::Type{S}, ::CodeInfo)::CodeInfo`. This default definition is
# simply the identity. Downstream code can overload this `pass` function
# for new context types.
pass(::Type{C}) where {C} = (signature, method_body) -> method_body

# Return the CodeInfo method body for type signature `S` and the world age `world`,
# if such a method exists in the method table. Otherwise, return `nothing`.
function lookup_method_body(::Type{S}, world::UInt) where {S<:Tuple}
    # ...
end

# Return `method_body` with every call expression wrapped in an `Overdub` carrying the
# same context and world as the caller (i.e. `f` in the `@generated` definition below).
# By convention, this caller is referenced in `CodeInfo` as `SlotNumber(1)`.
function overdub_calls!(method_body::CodeInfo)
    # ...
end

# This is the call definition for an `Overdub` wrapper.
@generated function (f::Overdub{F,C,world}})(args...) where {F,C,world}
    signature = Tuple{F,args...}
    method_body = lookup_method_body(signature, world)
    if isa(method_body, CodeInfo)
        # The `method_body` exists as `CodeInfo`, so we run the context's pass on
        # `method_body` and then wrap all downstream calls with `Overdub` instances
        # carrying the same context and world as `f`.
        method_body = overdub_calls!(pass(C)(signature, method_body))
    else
        # There is no retrievable method body, so just call the original function. This will
        # occur for Julia "built-in" functions, such as `getfield` or `arrayref`.
        method_body = :(f.func(args...))
    end
    return method_body
end
```

## Overdubbing vs. Method Overloading

As demonstrated above, the contextual passes injected by Cassette's overdubbing mechanism
are functions from a type signature and an original method body to a new method body.
Experienced Julia developers might note here that Julia actually already *has* a convenient
mechanism for emitting specialized method bodies from type signatures: method overloading
with multiple dispatch. So, then, what is the advantage of Cassette?

To answer this question, we need only examine the alternative pattern currently dominating
the Julia ecosystem: creating new argument types, and explicitly overloading methods on
these types. This pattern, while central to idiomatic Julia programming for most cases,
exhibits a few significant problems when applied as an alternative to Cassette's overdubbing
mechanism:

- *New subtypes must implement an informal/unchecked interface of their supertype,
regardless of the interface's (ir)relevance to the desired contextual transformation.* This
problem is evident when defining a new subtype requires implementing complex
conversion/promotion rules. These rules can rely on [subtle, unchecked behaviors](https://github.com/JuliaLang/julia/issues/17559),
and the required method overloads are not always evident given only a supertype. For a
real-world example, see [ForwardDiff's `Dual` number implementation](https://github.com/JuliaDiff/ForwardDiff.jl/blob/dd692d2f5c8014167a4d85c31d10d834361887fb/src/dual.jl#L306).

- *Type constraints in target programs drastically limit the domain of applicable code.*
Overly-strict type constraints are unavoidable in real-world code. Users and developers
often define methods with overly-strict type signatures to simplify development, or to
prevent code misuse. For example, if the goal was to intercept numeric code, then one
might define
    ```julia
    struct Foo{T<:Number} <: Number
        x::T
    end
    ```
    and intercept methods on `Foo`. Any methods of the form `f(::AbstractFloat)`, then,
    would be un-interceptable. Furthermore, even when one has access to target code and
    is allowed to modify it, post-hoc refactoring to loosen type restrictions can be
    quite an arduous task, and can even increase future maintenance burden (since future
    code must also be written generically). For a real-world example, see
    [JuliaStats/Distributions.jl#511](https://github.com/JuliaStats/Distributions.jl/pull/511).

- *Naive application of the pattern results in difficult-to-resolve dispatch ambiguities.*
More accurately, module-local information is not sufficient to easily discover and resolve
ambiguities between two different subtypes of the same supertype when extending a
multi-arity method on the supertype. Take the following example, imagining that
all three modules were written by different authors, and that`MA`'s author and
`MB`'s author are unaware of each other:
    ```julia
    module M
        abstract type T end
        f(::T, ::T) = "hello"
    end

    module MA
        using Main.M
        struct A <: M.T end
        M.f(::A, ::A)= 1
        M.f(::A, ::M.T) = 1
        M.f(::M.T, ::A) = 1
    end

    module MB
        using Main.M
        struct B <: M.T end
        M.f(::B, ::B) = 2
        M.f(::B, ::M.T) = 2
        M.f(::M.T, ::B) = 2
    end
    ```
    In practice, this scenario can commonly when two non-Base packages each implement a subtype
    of the same Base supertype and extend the same Base method as part of implementing the
    supertype's interface.

    If a user tried to compose these modules, they'd be in for a nasty surprise:

    ```julia
    julia> M.f(MA.A(), MB.B())
    ERROR: MethodError: Main.M.f(::Main.MA.A, ::Main.MB.B) is ambiguous. Candidates:
      f(::Main.M.T, ::Main.MB.B) in Main.MB at REPL[3]:6
      f(::Main.MA.A, ::Main.M.T) in Main.MA at REPL[2]:5
    Possible fix, define
      f(::Main.MA.A, ::Main.MB.B)
    ```

    For `Number` types, Julia's solution to this problem is to apply a map from the multiple
    dispatch case to the single dispatch case via it's conversion/promotion mechanism. However,
    this mechanism is only sensible in regimes where conversion between subtypes is cheap
    and well-defined.

    For a real-world example of this problem, see [JuliaDiff/ReverseDiff.jl#64](https://github.com/JuliaDiff/ReverseDiff.jl/issues/64).

- *Even with the aid of metaprogramming, explicit method overloading can only reflect on
extant types/methods, potentially causing problems due to load order dependencies.* Imagine
that the hypothetical module authors in our previous scenario were aware of this ambiguity
problem, but still not aware of each other. They might then attempt to resolve the
ambiguities via a bit of metaprogramming:
    ```julia
    module MA
        using Main.M
        struct A <: M.T end
        for S in (M.T, subtypes(M.T)...)
            @eval begin
                M.f(::A, ::A)= 1
                M.f(::A, ::$S) = 1
                M.f(::$S, ::A) = 1
            end
        end
    end

    module MB
        using Main.M
        struct B <: M.T end
        for S in (M.T, subtypes(M.T)...)
            @eval begin
                M.f(::B, ::B)= 2
                M.f(::B, ::$S) = 2
                M.f(::$S, ::B) = 2
            end
        end
    end
    ```
    This solution, while naively appealing since it gets rid of the error, is actually in some
    sense worse than the original problem. If one loads `MA` before `MB`, then
    `M.f(MA.A(), MB.B()) == 2`. If one loads `MB` before `MA`, then `M.f(MA.A(), MB.B()) == 1`.
    In other words, downstream code becomes "silently" load-order dependent!

- *Important built-in methods are not overloadable (e.g. `getfield`, `arrayset`).*

- *Functions of variable arity are non-trivial to intercept correctly.* See, for example,
[denizyuret/Knet.jl#112](https://github.com/denizyuret/Knet.jl/issues/112).

<!-- "Metadata determination" must be computed across arguments, which can be ambiguous/costly TODO: Move this problem to metadata section -->

In summary, Cassette's advantage over Julia's built-in method overloading is that it allows
context-specific method body transformations to proliferate throughout *any* code
running in the contextual "environment" without requiring explicit overloads to occur
for all downstream function calls.

## Contextual Dispatch

As an alternative to implementing overdub passes directly, Cassette provides an additional
dispatch layer on top of Julia's existing multiple dispatch mechanism. This "contextual
dispatch" layer enables the use of Julia's normal method overloading semantics to inject
new behaviors, while solving many of the method overloading problems listed in the previous
section. When possible, Cassette's contextual dispatch can be far easier and safer to use
than implementing the equivalent code transformation pass by hand.

## Dispatch Granularity and Contextual Primitives

# Cassette's Contextual Metadata Propagation

## Extending the Overdubbing Mechanism

## Avoiding Metadata Confusion

## Lifting Over Type Constraints