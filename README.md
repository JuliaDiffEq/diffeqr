# diffeqr

[![Build Status](https://travis-ci.org/JuliaDiffEq/diffeqr.svg?branch=master)](https://travis-ci.org/JuliaDiffEq/diffeqr)

diffeqr is a package for solving differential equations in R. It utilizes 
[DifferentialEquations.jl](http://docs.juliadiffeq.org/latest/) for its core routines 
to give high performance solving of ordinary differential equations (ODEs),
stochastic differential equations (SDEs), delay differential equations (DDEs), and
differential-algebraic equations (DAEs) directly in R.

## Installation

diffeqr is currently not registered into CRAN. Thus to use this package, use the following command:

```R
devtools::install_github('JuliaDiffEq/diffeqr', build_vignettes=T)
```

## Usage

diffeqr does not provide the full functionality of DifferentialEquations.jl. Instead, it supplies simplified
direct solving routines with an R interface. The most basic function is:

```R
ode.solve(f,u0,tspan,[p,abstol,reltol,saveat])
```

which solves the ODE `u' = f(u,p,t)` where `u(0)=u0` over the timespan `tspan`. 
The common interface arguments are documented 
[at the DifferentialEquations.jl page](http://docs.juliadiffeq.org/latest/basics/common_solver_opts.html).
Notice that not all options are allowed, but the most common arguments are supported.

## ODE Examples

### 1D Linear ODEs

Let's solve the linear ODE `u'=1.01u`. Start by loading the library:

```R
library(diffeqr)
```

Then define our derivative function `f(u,p,t)`. 

```R
f <- function(u,p,t) {
  return(1.01*u)
}
```

Then we give it an initial condition and a time span to solve over:

```R
u0 = 1/2
tspan <- list(0.0,1.0)
```

With those pieces we call `ode.solve` to solve the ODE:

```R
sol = ode.solve(f,u0,tspan)
```

This gives back a solution object for which `sol$t` are the time points
and `sol$u` are the values. We can check it by plotting the solution:

```R 
plot(sol$t,sol$u,"l")
```

![linear_ode](https://user-images.githubusercontent.com/1814174/39011970-e04f1fe8-43c7-11e8-8da3-848362691783.png)

### Systems of ODEs

Now let's solve the Lorenz equations. In this case, our initial condition is a vector and our derivative functions
takes in the vector to return a vector (note: arbitrary dimensional arrays are allowed). We would define this as:

```R
f <- function(u,p,t) {
  du1 = p[1]*(u[2]-u[1])
  du2 = u[1]*(p[2]-u[3]) - u[2]
  du3 = u[1]*u[2] - p[3]*u[3]
  return(c(du1,du2,du3))
}
```

Here we utilized the parameter array `p`. Thus we use `ode.solve` like before, but also pass in parameters this time:

```R
u0 = c(1.0,0.0,0.0)
tspan <- list(0.0,100.0)
p = c(10.0,28.0,8/3)
sol = ode.solve(f,u0,tspan,p=p)
```

The returned solution is like before. It is convenient to turn it into a data.frame:

```R
udf = as.data.frame(sol$u)
```

Now we can use `matplot` to plot the timeseries together:

```R
matplot(sol$t,udf,"l",col=1:3)
```

![timeseries](https://user-images.githubusercontent.com/1814174/39012314-ef7a8fe2-43c8-11e8-9dde-1a8b87d3cfa4.png)

Now we can use the Plotly package to draw a phase plot:

```R
library(plotly)
plot_ly(udf, x = ~V1, y = ~V2, z = ~V3, type = 'scatter3d', mode = 'lines')
```

![plotly_plot](https://user-images.githubusercontent.com/1814174/39012384-27ee7262-43c9-11e8-84d2-1edf937288ae.png)

Plotly is much prettier!

### Option Handling

If we want to have a more accurate solution, we can send `abstol` and `reltol`. Defaults are `1e-6` and `1e-3` respectively.
Generally you can think of the digits of accuracy as related to 1 plus the exponent of the relative tolerance, so the default is
two digits of accuracy. Absolute tolernace is the accuracy near 0. 

In addition, we may want to choose to save at more time points. We do this by giving an array of values to save at as `saveat`.
Together, this looks like:

```R
abstol = 1e-8
reltol = 1e-8
saveat = 0:10000/100
sol = ode.solve(f,u0,tspan,p=p,abstol=abstol,reltol=reltol,saveat=saveat)
udf = as.data.frame(sol$u)
plot_ly(udf, x = ~V1, y = ~V2, z = ~V3, type = 'scatter3d', mode = 'lines')
```

![precise_solution](https://user-images.githubusercontent.com/1814174/39012651-e03124e6-43c9-11e8-8496-bbee87987a37.png)

We can also choose to use a different algorithm. The choice is done using a string that matches the Julia syntax. See
[the ODE tutorial for details](http://docs.juliadiffeq.org/latest/tutorials/ode_example.html#Choosing-a-Solver-Algorithm-1).
The list of choices for ODEs can be found at the [ODE Solvers page](http://docs.juliadiffeq.org/latest/solvers/ode_solve.html).
For example, let's use a 9th order method due to Verner:

```R
sol = ode.solve(f,u0,tspan,alg="Vern9()",p=p,abstol=abstol,reltol=reltol,saveat=saveat)
```

Note that each algorithm choice will cause a JIT compilation.

## Performance Enhancements

One way to enhance the performance of your code is to define the function in Julia so that way it is JIT compiled. diffeqr is
built using [the JuliaCall package](https://github.com/Non-Contradiction/JuliaCall), and so you can utilize this same interface
to define a function directly in Julia:

```R
f <- julia_eval("
function f(du,u,p,t)
  du[1] = 10.0*(u[2]-u[1])
  du[2] = u[1]*(28.0-u[3]) - u[2]
  du[3] = u[1]*u[2] - (8/3)*u[3]
end")
```

We can then use this in our ODE function by telling it to use the Julia-defined function called `f`:

```R
u0 = c(1.0,0.0,0.0)
tspan <- list(0.0,100.0)
sol = ode.solve(f,u0,tspan,fname="f")
```

This will help a lot if you are solving difficult equations (ex. large PDEs) or repeat solving (ex. parameter estimation).

## Systems of SDEs

Solving stochastic differential equations (SDEs) is the similar. To solve a diagonal noise SDE, you use `sde.solve` and give
two functions: `f` and `g`, where `du = f(u,t)dt + g(u,t)dW_t`. For example, let's add diagonal multiplicative noise to the
Lorenz attractor:

```R
f <- function(u,p,t) {
  du1 = p[1]*(u[2]-u[1])
  du2 = u[1]*(p[2]-u[3]) - u[2]
  du3 = u[1]*u[2] - p[3]*u[3]
  return(c(du1,du2,du3))
}
g <- function(u,p,t) {
  return(c(0.3*u[1],0.3*u[2],0.3*u[3]))
}
u0 = c(1.0,0.0,0.0)
tspan <- list(0.0,1.0)
p = c(10.0,28.0,8/3)
sol = sde.solve(f,g,u0,tspan,p=p,saveat=0.005)
udf = as.data.frame(sol$u)
plotly::plot_ly(udf, x = ~V1, y = ~V2, z = ~V3, type = 'scatter3d', mode = 'lines')
```

Using a JIT compiled function for the drift and diffusion functions can greatly enhance the speed here.
With the speed increase we can comfortably solve over long time spans:

```R
f <- julia_eval("
function f(du,u,p,t)
  du[1] = 10.0*(u[2]-u[1])
  du[2] = u[1]*(28.0-u[3]) - u[2]
  du[3] = u[1]*u[2] - (8/3)*u[3]
end")

g <- julia_eval("
function g(du,u,p,t)
  du[1] = 0.3*u[1]
  du[2] = 0.3*u[2]
  du[3] = 0.3*u[3]
end")
tspan <- list(0.0,100.0)
sol = sde.solve(f,g,u0,tspan,fname="f",gname="g",p=p,saveat=0.05)
udf = as.data.frame(sol$u)
#plotly::plot_ly(udf, x = ~V1, y = ~V2, z = ~V3, type = 'scatter3d', mode = 'lines')
```

![stochastic_lorenz](https://user-images.githubusercontent.com/1814174/39019723-216c3210-43df-11e8-82c0-2e676f53e235.png)
