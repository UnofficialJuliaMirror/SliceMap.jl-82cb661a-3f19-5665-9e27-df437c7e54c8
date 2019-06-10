# SliceMap.jl

[![Build Status](https://travis-ci.org/mcabbott/SliceMap.jl.svg?branch=master)](https://travis-ci.org/mcabbott/SliceMap.jl)

This package provides some `mapslices`-like functions, 
with gradients for [Flux](https://github.com/FluxML/Flux.jl) and [Zygote](https://github.com/FluxML/Zygote.jl):

```julia
mapcols(f, M) ≈ mapreduce(f, hcat, eachcol(M))
MapCols{d}(f, M)         # where d=size(M,1), for SVector slices
ThreadMapCols{d}(f, M)   # using Threads.@threads

maprows(f, M) ≈ mapslices(f, M, dims=2)

slicemap(f, A; dims) ≈ mapslices(f, A, dims=dims) # only Zygote
```

<!--
It also defines Zygote gradients for the Slice/Align functions in 
[JuliennedArrays](https://github.com/bramtayl/JuliennedArrays.jl), 
and the slice/glue functions in [TensorCast](https://github.com/mcabbott/TensorCast.jl), 
both of which are good ways to roll-your-own `mapslices`-like behaviour.
-->

### Simple example

```julia
mat = rand(1:9, 3,10)
fun(x) = 2 .+ x.^2
mapslices(fun, mat, dims=1)

using SliceMap
mapcols(fun, mat)     # eachcol(m)
MapCols{3}(fun, mat)  # reinterpret(SArray,...)

using ForwardDiff, Tracker, Zygote
ForwardDiff.gradient(m -> sum(sin, mapslices(fun, m, dims=1)), mat)

Tracker.gradient(m -> sum(sin, mapcols(fun, m)), mat)[1]     # Tracker.forward per slice
Tracker.gradient(m -> sum(sin, MapCols{3}(fun, m)), mat)[1]  # ForwardDiff on slices

Zygote.gradient(m -> sum(sin, mapcols(fun, m)), mat)[1]      # Zygote.forward per slice
Zygote.gradient(m -> sum(sin, MapCols{3}(fun, m)), mat)[1]
```

These are a bit faster than `mapslices` too. Although storing all the backward functions, 
which is what `mapcols` does, seems not to be so quick:

```julia
using BenchmarkTools
mat1k = rand(3,1000);

@btime mapreduce(fun, hcat, eachcol($mat1k)) # 1.522 ms
@btime mapslices(fun, $mat1k, dims=1)        # 1.017 ms

@btime mapcols(fun, $mat1k)                  #   399.016 μs
@btime MapCols{3}(fun, $mat1k)               #    15.564 μs
@btime MapCols(fun, $mat1k)                  #    16.774 μs  without size

@btime ForwardDiff.gradient(m -> sum(sin, mapslices(fun, m, dims=1)), $mat1k); # 372.705 ms
@btime Tracker.gradient(m -> sum(sin, mapcols(fun, m)), $mat1k);               #  70.203 ms
@btime Tracker.gradient(m -> sum(sin, MapCols{3}(fun, m)), $mat1k);            #     146.561 μs, 330.51 KiB
@btime Zygote.gradient(m -> sum(sin, mapcols(fun, m)), $mat1k);                #  20.018 ms, 3.82 MiB
@btime Zygote.gradient(m -> sum(sin, MapCols{3}(fun, m)), $mat1k);             #     245.550 μs
```

### Other packages

This package also provides Zygote gradients for the slice/glue functions in 
[TensorCast](https://github.com/mcabbott/TensorCast.jl),
which can be used to write many mapslices-like operations.
(The function `slicemap(f, A, dims)` uses these functions, without having to write index notation.)

```julia
using TensorCast
@cast [i,j] := fun(mat[:,j])[i]                        # same as mapcols

tcm(mat) = @cast out[i,j] := fun(mat[:,j])[i]
Zygote.gradient(m -> sum(sin, tcm(m)), mat)[1]

@btime tcm($mat1k)                                     #    407.176 μs
@btime Zygote.gradient(m -> sum(sin, tcm(m)), $mat1k); # 19.086 ms
```

Similar gradients work for the Slice/Align functions in 
[JuliennedArrays](https://github.com/bramtayl/JuliennedArrays.jl),
so it defines these too:

```julia
using JuliennedArrays
jumap(f,m) = Align(map(f, Slices(m, True(), False())), True(), False())
jumap(fun, mat)                                               # same as mapcols
Zygote.gradient(m -> sum(sin, jumap(fun, m)), mat)[1]

@btime jumap(fun, $mat1k);                                    #    408.259 μs
@btime Zygote.gradient(m -> sum(sin, jumap(fun, m)), $mat1k); # 18.638 ms
```

That's a 2-line gradient definition, so borrowing it may be easier than depending on this package. 

The original purpose of `MapCols`, with ForwardDiff on slices, was that this works well when
the function being mapped integrates some differential equation. 

```julia
using DifferentialEquations, ParameterizedFunctions
ode = @ode_def begin
  du = ( - k2 * u )/(k1 + u) # an equation with 2 parameters
end k1 k2

function g(k::AbstractVector{T}, times) where T
    u0 = T[ 1.0 ] # NB convert initial values to eltype(k)
    prob = ODEProblem(ode, u0, (0.0, 0.0+maximum(times)), k)
    Array(solve(prob, saveat=times))::Matrix{T}
end

kay = rand(2,50);
MapCols{2}(g, kay, 1:5) # 5 time steps, for each col of parameters

Tracker.gradient(k -> sum(sin, MapCols{2}(g, k, 1:5)), kay)[1]
```

This is both quite efficient, and seems to go well with multi-threading:

```julia
@btime MapCols{2}(g, $kay, 1:5)        # 1.369 ms
@btime ThreadMapCols{2}(g, $kay, 1:5)  #   670.384 μs

@btime Tracker.gradient(k -> sum(sin, MapCols{2}(g, k, 1:5)), $kay)[1]       # 2.438 ms
@btime Tracker.gradient(k -> sum(sin, ThreadMapCols{2}(g, k, 1:5)), $kay)[1] # 1.229 ms

Threads.nthreads() == 4
```

### Elsewhere

Issues about mapslices:
* https://github.com/FluxML/Zygote.jl/issues/92
* https://github.com/FluxML/Flux.jl/issues/741
* https://github.com/JuliaLang/julia/issues/29146

Differential equations:
* https://arxiv.org/abs/1812.01892 "DSAAD"
* http://docs.juliadiffeq.org/latest/analysis/sensitivity.html

Other packages which define gradients of possible interest:
* https://github.com/GiggleLiu/LinalgBackwards.jl
* https://github.com/mcabbott/ArrayAllez.jl

Differentiation packages this could perhaps support, quite the zoo:
* https://github.com/dfdx/Yota.jl
* https://github.com/invenia/Nabla.jl
* https://github.com/denizyuret/AutoGrad.jl
* https://github.com/Roger-luo/YAAD.jl
* And perhaps one day, just https://github.com/JuliaDiff/ChainRules.jl
