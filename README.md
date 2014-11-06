# Jags


[![Jags](http://pkg.julialang.org/badges/Jags_release.svg)](http://pkg.julialang.org/?pkg=Jags&ver=release)

## Purpose

A package to use Jags (as an external program) from Julia.

For more info on Jags, please go to <http://mcmc-jags.sourceforge.net>.

For more info on Mamba, please go to <http://mambajl.readthedocs.org/en/latest/>.

This version will be kept as the Github branch Jags-j0.3-v0.1.0.


## What's new

### Version 0.1.0

The two most important features introduced in version 0.1.0 are:

1. Using Mamba to display and diagnose simulation results. The call to jags() to sample now returns a Mamba Chains object (previously it returned a dictionary). 
2. Added the ability to specify RNGs in the initializations file for running simulations in parallel.

### Version 0.0.4

1. Added the ability to start multiple Jags scripts in parallel. Work in progress.

### Version 0.0.3 and earlier

1. Parsing structure for input arguments to Stan.
2. Single process execution of a Jags simulations.
3. Read created output files by Jags back into Julia.


## Requirements

This version of the Jags.jl package assumes that: 

1. Jags is installed and the jags binary is on $PATH.

2. Mamba (see <https://github.com/brian-j-smith/Mamba.jl>) is installed. It can be installed using Pkg.add("Mamba")

3. On OSX, all Jags-j03-v0.1.0 examples check the environment variable JULIA_SVG_BROWSER to automatically display (in a browser) the simulation results (after creating .svg files), e.g. on my system I have exported JULIA_SVG_BROWSER="Google Chrome.app". For other platforms the final lines in the Examples/xxxx.jl files may need to be adjusted (or removed). In any case, on all platforms, both a .svg and a .pdf file will be created and left behind in the working directory.

This version of the package has primarily been tested on Mac OSX 10.10, Julia 0.3.2, Jags 3.4.0 and Mamba 0.3.9. A limited amount of testing has taken place on other platforms by other users of the package (see note 1 in the 'To Do' section below).

To test and run the examples:

**julia >** ``Pkg.test("Jags")``


## A walk through example

As the Jags program produces results files in a subdirectory of the current directory, it is useful to control the current working directory and restore the original directory at the end of the script (as is demonstrated in the test_xxx.jl scripts in the test subdirectory).
```
using Compat, Mamba, Jags

old = pwd()
ProjDir = Pkg.dir("Jags", "Examples", "Line1")
cd(ProjDir)
```
Variable `line` holds the model which will be writtten to a file named `$(model_name).bugs`. The value of model_name is set later on, see Jagsmodel().
```
line = "
model {
  for (i in 1:n) {
        mu[i] <- alpha + beta*(x[i] - x.bar);
        y[i]   ~ dnorm(mu[i],tau);
  }
  x.bar   <- mean(x[]);
  alpha    ~ dnorm(0.0,1.0E-4);
  beta     ~ dnorm(0.0,1.0E-4);
  tau      ~ dgamma(1.0E-3,1.0E-3);
  sigma   <- 1.0/sqrt(tau);
}
"
```
Next, define which variables should be monitored (if => true).
```
monitors = (ASCIIString => Bool)[
  "alpha" => true,
  "beta" => true,
  "tau" => true,
  "sigma" => true,
]
```
The next step is to create and initialize a Jagsmodel:
```
jagsmodel = Jagsmodel(
  name="line1", 
  model=line,
  monitor=monitors,
  #ncommands=1, nchains=4,
  #deviance=true, dic=true, popt=true,
  pdir=ProjDir);

println("\nJagsmodel that will be used:")
jagsmodel |> display
```
Notice that by default a single commands with 4 chains is created. It is possible to run each of the 4 chains in a separate process which has advantages. Using the Bones example as a testcase, on my machine running 1 command simulating a single chain takes 6 seconds, 4 (parallel) commands each simulating 1 chain takes about 9 seconds and a single command simulating 4 chains takes about 25 seconds. Of course this is dependent on the number of available cores and assumes the drawing of samples takes a reasonable chunk of time vs. running a command in a new shell.

Running chains in separate commands does need additional data to be passed in through the initialization data and is demonstrated in Examples/Line2.

If nchains is set to 1, this is updated in Jagsmodel if DIC and/or popt is requested. Jags needs minimally 2 chains to compute those.

The input data for the line examples are in below data dictionary using the @Compat macro for compatibility between Julia v0.3 and v0.4:
```
data = @Compat.Dict(
  "x" => [1, 2, 3, 4, 5],
  "y" => [1, 3, 3, 3, 5],
  "n" => 5
)

println("Input observed data dictionary:")
data |> display
```
Next define an array of dictionaries with initial values for parameters. If the array of dictionaries has not enough elements, the elements will be recycled for chains/commands:
```
inits = [
  @Compat.Dict("alpha" => 0,"beta" => 0,"tau" => 1),
  @Compat.Dict("alpha" => 1,"beta" => 2,"tau" => 1),
  @Compat.Dict("alpha" => 3,"beta" => 3,"tau" => 2),
  @Compat.Dict("alpha" => 5,"beta" => 2,"tau" => 5)
]
inits = map((x)->convert(Dict{ASCIIString, Any}, x), inits)

println("\nInput initial values dictionary:")
inits |> display
println()
```
Run the mcmc simulation, passing in the model, the data, the initial values and the working directory. If 'inits' is a single dictionary, it needs to be passed in as '[inits]', see the Bones example. 
```
sim = jags(jagsmodel, data, inits, ProjDir)
describe(sim)
println()
```

## Running a Jags script, some details

Jags.jl really only consists of 2 functions, Jagsmodel() and jags().

Jagsmodel() is used to define and set up the basic structure to run a simulation.
The full signature of Jagsmodel() is:
```
function Jagsmodel(;
  name="Noname", 
  model="", 
  ncommands=1,
  nchains=4,
  adapt=1000,
  update=10000,
  thin=10,
  monitor=Dict(), 
  deviance=false,
  dic=false,
  popt=false,
  updatejagsfile=true,
  pdir=pwd())
```
All arguments are keyword arguments and have default values, although usually at least the name and model arguments will be provided.

After a Jagsmodel has been created, the workhorse function jags() is called to run the simulation, passing in the Jagsmodel, the data and the initialization for the chains.

As Jags needs quite a few input files and produces several output files, these are all stored in a subdirectory of the working directory, typically called 'tmp'.

The full signature of jags() is:
```
function jags(
  model::Jagsmodel,
  data::Dict{ASCIIString, Any}=Dict{ASCIIString, Any}(),
  init::Array{Dict{ASCIIString, Any}, 1} = Dict{ASCIIString, Any}[],
  ProjDir=pwd();
  updatedatafile::Bool=true,
  updateinitfiles::Bool=true
  )
```
All parameters to compile and run the Jags script are implicitly passed in through the model argument.

The Line2 example shows how to run multiple Jags simulations in parallel. In the most simple case, e.g. 4 commands, each with a single chain, can be initialized with an 'inits' like shown below:
```
inits = [
  @Compat.Dict("alpha" => 0,"beta" => 0,"tau" => 1,".RNG.name" => "base::Wichmann-Hill"),
  @Compat.Dict("alpha" => 1,"beta" => 2,"tau" => 1,".RNG.name" => "base::Marsaglia-Multicarry"),
  @Compat.Dict("alpha" => 3,"beta" => 3,"tau" => 2,".RNG.name" => "base::Super-Duper"),
  @Compat.Dict("alpha" => 5,"beta" => 2,"tau" => 5,".RNG.name" => "base::Mersenne-Twister")
]
```


## To do

More features will be added as requested by users and as time permits. Please file an issue/comment/request.

The ability to resume a simulation will also be looked at for version 0.1.x.

**Note 1:** In order to support platforms other than OS X, help is needed to test on such platforms.
