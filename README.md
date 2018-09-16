# Soss

Soss is a library for manipulating source-code representation of probabilistic models.

**Soss IS "PRE-ALPHA" SOFTWARE -- BREAKING CHANGES ARE IMMINENT**


## Dependency Graphs

```julia
julia> lda
@model (α, N, K, V, η) begin
    M = length(N)
    β ~ Dirichlet(repeat([η], V)) |> iid(K)
    θ ~ Dirichlet(repeat([α], K)) |> iid(M)
    z ~ For(1:M) do m
            Categorical(θ[m]) |> iid(N[m])
        end
    w ⩪ For(1:M) do m
            For(1:N[m]) do n
                Categorical(β[(z[m])[n]])
            end
        end
end

julia> g = graph(lda); [(g[e.src,:name] => g[e.dst,:name]) for e in edges(g)]
14-element Array{Pair{Symbol,Symbol},1}:
 :α => :θ
 :N => :w
 :N => :M
 :N => :z
 :M => :w
 :M => :z
 :M => :θ
 :K => :β
 :K => :θ
 :V => :β
 :z => :w
 :β => :w
 :η => :β
 :θ => :z
```
## Coming Soon

- Macro optimization of densities, as in [Passage](https://www.dropbox.com/s/zg2g0cfiin0jdmr/Scherrer%20et%20al.%20-%202014%20-%20Passage%20A%20Parallel%20Sampler%20Generator%20for%20Hierarchical%20Bayesian%20Modeling.pdf)
- Optimization based on exponential families, see [here](https://www.dropbox.com/s/26omxn6zo8gia3u/Scherrer%20-%20Unknown%20-%20An%20Exponential%20Family%20Basis%20for%20Probabilistic%20Programming.pdf?dl=0)

---
Stuff below this point is outdated, updated coming soon

## Old Docs

Here's a simple linear regression model in Soss:

```julia
linReg1D = @model N begin
    # Priors chosen following Gelman(2008)
    α ~ Cauchy(0,10)
    β ~ Cauchy(0,2.5)
    σ ~ Truncated(Cauchy(0,3), 0, Inf)
    x ~ For(1:N) do n 
        Cauchy(0,100)
    end
    ŷ = α + β .* x
    y ~ For(1:N) do n 
        Normal(ŷ[n], σ)
    end
end
```

This produces a Julia expression representing a function:
```julia
:(function (N,)
        α ~ Cauchy(0, 10)
        β ~ Cauchy(0, 2.5)
        σ ~ Truncated(Cauchy(0, 3), 0, Inf)
        x ~ For(((n,)->Cauchy(0, 100)), 1:N)
        ŷ = α + β .* x
        y ~ For(((n,)->Normal(ŷ[n], σ)), 1:N)
    end)
```

Note that the notation after parsing can be slightly different than it originally entered.

Usually, we're not interested in the distribution on `x`, so we condition on it. This removes the distribution declaration, instead passing it as a parameter:

```julia
Main> lr2 = condition(linReg1D, :x)
:(function (N, x)
        α ~ Cauchy(0, 10)
        β ~ Cauchy(0, 2.5)
        σ ~ Truncated(Cauchy(0, 3), 0, Inf)
        ŷ = α + β .* x
        y ~ For(((n,)->Normal(ŷ[n], σ)), 1:N)
    end)
```

We need to be careful with `condition` in cases where the distribution depends on other random values. The implementation will eventually take this into account, but it doens't yet.

From here, we can "run the model" forward, specifying `x` and generating both the parameters and the `y` response. To instead "observe" the `y` values, we need a different approach that conditional. To `observe` a quantity means it is passed as an argument, but the value still affects the distribution:

```julia
Main> lr3 = observe(lr2, :y)
:(function (N, x, y)
        α ~ Cauchy(0, 10)
        β ~ Cauchy(0, 2.5)
        σ ~ Truncated(Cauchy(0, 3), 0, Inf)
        ŷ = α + β .* x
        y <~ For(((n,)->Normal(ŷ[n], σ)), 1:N)
    end)
```

After building a model, you can query it:

```julia
> parameters(lr3)
3-element Array{Symbol,1}:
 :α
 :β
 :σ

Main> supports(lr3)
Dict{Symbol,Any} with 3 entries:
  :α => Distributions.RealInterval(-Inf, Inf)
  :σ => Distributions.RealInterval(0.0, Inf)
  :β => Distributions.RealInterval(-Inf, Inf)
```


Or you can transform it to a form suitable for specialized inference algorithms. For example, a Stan-like approach:

(works for older implementation, but for now this is just a mockup) 
```julia
> logdensity(lr3)
:(function ((N,x,y), θ)
        ℓ = 0.0
        α = θ[1]
        ℓ += logpdf(Cauchy(0, 10), α)
        β = θ[2]
        ℓ += logpdf(Cauchy(0, 2.5), β)
        σ = softplus(θ[3])
        ℓ += abs(σ - θ[3])
        ℓ += logpdf(Truncated(Cauchy(0, 3), 0, Inf), σ)
        ŷ = α + β .* x
        ℓ += logpdf(For(((n,)->Normal(ŷ[n], σ)), 1:N), y)
        return ℓ
    end)
```

## The name

* "Source" (the stuff transformed by Soss), said with a thick Northeastern accent
* Cockney rhyming slang ("sauce pan" rhymes with "[Stan](http://mc-stan.org/)")
* **S**oss is **O**pen **S**ource **S**oftware

