# AnimalBehavior.jl

[![Build status (Github Actions)](https://github.com/sqwayer/AnimalBehavior.jl/workflows/CI/badge.svg)](https://github.com/sqwayer/AnimalBehavior.jl/actions)
[![codecov.io](http://codecov.io/github/sqwayer/AnimalBehavior.jl/coverage.svg?branch=main)](http://codecov.io/github/sqwayer/AnimalBehavior.jl?branch=main)
[![](https://img.shields.io/badge/docs-stable-blue.svg)](https://sqwayer.github.io/AnimalBehavior.jl/stable)
[![](https://img.shields.io/badge/docs-dev-blue.svg)](https://sqwayer.github.io/AnimalBehavior.jl/dev)

Generative models of animal behavior rely on the same global structure : 
- They are defined by a set of ***latent variables*** (either fixed parameters or evolving variables)
- Those latent variables evolve as a function of external observations by an ***evolution function***
- Actions are generated by sampling from distributions defined by an ***observation function*** of the latent variables

AnimalBehavior.jl takes advantage of this common structure to wrap some functionnalities of the [Turing langage for dynamic probabilistic programming](https://github.com/TuringLang), in order to simulate and fit behavioral models with a minimal set of specifications from the user.


## Create a model
First, you need to create a [DynamicPPL model](https://github.com/TuringLang) using the ```@model``` macro, that returns all the latent variables of your model as a ```NamedTuple``` : 
```julia
@model Qlearning(na, ns) = begin
    α ~ Beta()
    logβ ~ Normal(1,1)

    return (α=α, β=exp(logβ), Values = fill(1/na,na,ns))
end

MyModel = Qlearning(2,1)

```

Latent variables can be sampled from a prior distribution, and/or transformed by any arbitrary function 

Then you have to define an evolution and an observation functions with the macros ```@evolution```and ```@observation```respectively with the following syntax : 
```julia
@evolution MyModel begin 
        Values[a,s] += α * (r - Values[a,s]) # or : delta_rule!(s, a, r, Values, α)
    end

@observation MyModel begin
        Categorical(softmax(β * @views(Values[:,s])))
    end
```

The expression in the ```begin``` ```end``` statement can use the **reserved variables names** ```s```, ```a``` and ```r``` for the current state, action and feedback respectively, and/or any latent variable defined earlier.
Moreover, the observation function must return a ```Distribution``` from the [Distributions.jl package](https://github.com/JuliaStats/Distributions.jl).

## Simulate behavior
```julia
# Simulation of a probabilistic reversal task
function pr_feedback(history) # Reverse the correct response every 20 trials
    correct = mod(length(history)/20, 2) < 1 ? 1 : 2
    return rand() < 0.9 ? history[end].a == correct : history[end].a ≠ correct 
end

sim = simulate(MyModel; feedback=pr_feedback);
```
```simulate``` returns a ```Simulation``` structure with fields ```data``` and ```latent```.

## Inference
The package re-export the ```sample``` function to return a [Chains](https://github.com/TuringLang/MCMCChains.jl) object, with the following syntax : 
```julia
sample(model, data, args...; kwargs...)
```
e.g. : 
```julia
chn = sample(MyModel, sim.data, NUTS(), 1000)
```

## Model comparison
WIP
