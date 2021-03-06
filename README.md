# CBOOCall

[![Build Status](https://github.com/jlapeyre/CBOOCall.jl/actions/workflows/CI.yml/badge.svg?branch=main)](https://github.com/jlapeyre/CBOOCall.jl/actions/workflows/CI.yml?query=branch%3Amain)
[![Coverage](https://codecov.io/gh/jlapeyre/CBOOCall.jl/branch/main/graph/badge.svg)](https://codecov.io/gh/jlapeyre/CBOOCall.jl)

This package provides `@cbooify` which allows you to write the function call `f(a::A, args...)` as `a.f(args...)` as well.
You can use it by adding a single line to your module. Using the alternative call syntax incurs no performance
penalty.

The main motivation is to make it easy to call many functions with short names without bringing
them into scope. For example `s.x(1)`, `s.y(3)`,  `s.z(3)`, etc. We want to do this without
claiming `x`, `y`, `z`, among others. This is all the package does.

For example, in building  quantum computing circuits programmatically, people really want
to write `circ.x(1)` to add an `X` gate on wire `1`. You could do
```julia
using QCircuit: add!  # ok to import this probably

add!(circ, QCircuit.x, 1) # But, I really don't want to import x, y, z, etc.
```

Here is [an example](https://github.com/rafal-pracht/QuantumCircuits.jl/blob/b1463aa6aac3c088c3ca14b90067a525788ddf8b/src/QCircuits/Circuit.jl#L93) from an application
```julia
@cbooify QCircuit (x, sx, y, z, h, cx, s, sdg, t, tdg, u, u3, rx, ry, rz, rzx, u4, barrier, measure)
```

This package doesn't offer other features of typical OO systems. This package writes a `getproperty`
method. So any other OO features that need to be in `getproperty` might go in CBOOCall.jl

A requirement of design is no performance penalty. I mean I would not have brought
this to a package if there were a performance penalty.
And benchmarking the code in the test suite shows there is none.
But, there may be some lurking somewhere.

For example the script [smalltest.jl](./smalltest.jl)
```julia
push!(LOAD_PATH, "./test/MyAs/", "./test/MyBs/")
using MyAs
using BenchmarkTools
const y2 = MyA(5)
@btime [y2.sx(i) for i in 1:100];
@btime [MyAs.sx(y2, i) for i in 1:100];
```
Prints:
```
  40.218 ns (1 allocation: 896 bytes)
  40.025 ns (1 allocation: 896 bytes)
```

#### Usage

```julia
module Amod

using CBOOCall: @cbooify

struct A
  x::Int
end

@cbooify A (f, g)

f(a::A, x, y) = a.x + x + y
g(a::A) = a.x

end # module Amod
```

Then you can write either `Amod.f(a, 1, 2)` or `a.f(1, 2)`.

For more features and details, see the docstring.

#### Functions and macros
`@cbooify`, `add_cboo_calls`, `is_cbooified`, `whichmodule`, `cbooified_properties`.

#### Docstring

    @cbooify(Type_to_cbooify, (f1, f2, fa = Mod.f2...), callmethod=nothing, getproperty=getfield)

Allow functions of the form `f1(s::Type_to_cbooify, args...)` to also be called with `s.f1(args...)` with no performance penalty.

`callmethod` and `getproperty` are keyword arguments.

If an element of the `Tuple` is an assignment `sym = func`, then `sym` is the property
that will call `func`. `sym` must be a simple identifier (a symbol). `func` is not
required to be a symbol. For example `myf = Base._unexportedf`.

If `callmethod` is supplied, then `s.f1(args...)` is translated to `callmethod(s, f1,
args...)` instead of `f1(s, args...)`.

`@cbooify` works by writing methods (or clobbering methods) for the functions
`Base.getproperty` and `Base.propertnames`.

`getproperty` must be a function. If supplied, then it is called, rather than `getfield`, when looking up a
property that is not on the list of functions. This can be useful if you want further
specialzed behavior of `getproperty`.

`@cbooify` must by called after the definition of `Type_to_cbooify`, but may
be called before the functions are defined.

If an entry is not function, then it is returned, rather than called.  For example
`@cbooify MyStruct (y=3,)`. Callable objects meant to be called must be wrapped in a
function.

#### Examples:

* Use within a module

```julia
module Amod
import CBOOCall

struct A
    x::Int
end

CBOOCall.@cbooify A (w, z)

w(a::A, y) = a.x + y
z(a::A, x, y) = a.x + y + x
end # module
```
```julia-repl
julia> a = Amod.A(3);

julia> Amod.w(a, 4) == a.w(4) == 7
true

julia> CBOOCall.whichmodule(a)
Main.Amod

julia> CBOOCall.cboofied_properties(a)
(w = Main.Amod.w, z = Main.Amod.z)
```

* The following two calls have the same effect.

```julia
@cbooify(Type_to_cbooify, (f1, f2, ...))

@cbooify(Type_to_cbooify, (f1, f2, ...), callmethod=nothing, getproperty=getfield)
```

<!--  LocalWords:  CBOOCall args Benchmarking smalltest jl julia MyAs const MyA sx
 -->
<!--  LocalWords:  BenchmarkTools btime ns Amod cboo struct docstring
 -->
