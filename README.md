# VIDA

[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://ptiede.github.io/VIDA.jl/dev)
[![Build Status](https://travis-ci.com/ptiede/VIDA.jl.svg?branch=master)](https://travis-ci.com/ptiede/VIDA.jl)
[![Coverage](https://codecov.io/gh/ptiede/VIDA.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/ptiede/VIDA.jl)


**Now a registered Julia package use ]add VIDA to install in local Julia environment**

VIDA.jl or the *Variational Image Domain Analysis* provides a interface to extracting features from fits images created for the EHT, using the notion of probabilty divergences similar to variational inference, hence the name. The currently implemented divergences are the Bhattacharyya distance/divergence as well as the Kullback-Leiber divergence. These are used to extract ring-like features from image reconstructions of black holes such as from M87. A paper applying this to various images ring-like images is in preparation.

## Interface
The inteface is based off of using probability divergences to extract features from images. The main functions a user needs to be aware of are:

``` julia
#load the image
image = load_ehtimfits(file::String)
#create the divergence we want (Bhattacharyya divergence)
bh = Bhattacharyya(image)
#Create the filter we want to use to extract for example an slashed elliptical Gaussian
filter = GeneralGaussianRing(p::Array) <: AbstractFilter
#To extract the features we use the extract function
extract(bh,
          θ<:AbstractFilter, #starting location of filter
          lower,  #lower bounds on parameters for filter
          upper;   #upper bounds on the parameters for filter
          method=Fminbox(LBFGS()) #optimization method
         )
#To use a more robust global optimizer use
bbextract(bh,
          θ::AbstractFilter, #starting location of filter
          lower,  #lower bounds on parameters for filter
          upper;   #upper bounds on the parameters for filter
          MaxFuncEvals = 2*10^4, #Maximum number of measure evals
          TraceMode = :compact # write progress
         )
```
Let's dive into what each piece means

### Image type: `EHTImage`
 - This is the image you want to fit. Currently we only have support for fits images that are similar to the fits image objects created by [ehtim](https://github.com/achael/eht-imaging). These images are loaded with the function
 ```julia
  #load fits image
  image = load_ehtimfits("fitsname.fits")
```
 - There are an additional number of tools available for image processing, such as clipping flux and increasing constrast in the image. That will be included in the documentation when I get around to making it.

### Filter type `AbstractFilter`
 - This is the filter that will be used to extract an image feature. Basically it will find the filter that is the closest to. Currently there are 4 types of filters implemented, but each are a subset of the other:
```julia
    GaussianRing(p) #Gaussian circular ring
    SlashedGaussianRing(p) #Gaussian circular ring with flux asymmetry
    EllipticalGaussianRing(p) #Gaussian elliptical ring
    TIDAGaussianRing(p)  #Gaussian elliptical ring with flux asymmetry where the orientations relative to each other are fixed
    GeneralGaussianRing(p) #Gaussian elliptical slashed ring where orientation of the slash and asymmetry are independent.
```
 - For the parameters that it takes please use the julia ? mode.

 If you want to add your own filter you just need to define a new Filter type a size method, and a imagefilter method for that type of Filter. For example if you want to add a symmetric Gaussian filter you would just add
 ```julia
@with_kw struct SymGaussian <: AbstractFilter
    σ::Float64 #standard deviation of the Gaussian
    x0::Float64 #x location of mean
    y0::Float64 #y location of mean
end

SymGaussian(p) = SymGaussian(p[1],p[2],p[3])

size(::Type{SymGaussian}) = 3

function (θ::SymGaussian)(x,y)
   z2 = ((x-θ.x0)^2 + (x-θ.y0)^2)/(2.0*θ.^2)
   return = 1.0/(2π*θ.σ^2)*exp(-z2)
end
 ```
Then you can simply call the same optimizing functions and plotting functions. Pretty neat eh?

You know what is even cooler? You can add filters together, and multiply then by a number. For example to plot a added filter just use
```julia
θ1 = GaussianRing(15.0,10.0,1.0,1.0)
θ2 = SymGaussian(5,-5,-5)
θ = θ1 + 5*θ2

plot(θ)
```
The plotting is done through the recipes macros in Plots.jl. So it should 
just work! In addition to the plot function there is a new recipe called
`triptic(img,θ)` that will produce a comparisson between the filter and
the true image. This can be useful when comparing the best filter to the 
image.


Additionally any other function that dispatches on the filter type should just work! One thing to note is that the weight between the two filters is relative. Namely, total intensity will always be normalized, so the above code says that θ2 has 5 times the relative flux compared to the first.


### Divergence
In order to extract a feature you need to create a probability divergence function. Currently the divergences are defined using a AbstractDivergence type. Currently we have two divergences implemented [Bhattacharyya divergence (Bh)](https://en.wikipedia.org/wiki/Bhattacharyya_distance) and the [Kullback-Leiber divergence (KL)](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence). In order to construct the divergence we first need to specify the `image` that we are trying to fit. 
```julia
bh = Bhattacharyya(image) #make the Bh divergence
kl = KullbackLeibler(image) #makes the KL divergence
```
which creates a functor that depends on the image. The functor itself take a filter, e.g.,
```julia
bh(θ::AbstractFilter)
```
and `bh` will use multiple dispatch to figure out which filter function to use.


### Extract `extract` and `bbextract`
We then use the extract function to extract the image feature. Currently this uses either [Optim.jl](https://github.com/JuliaNLSolvers/Optim.jl) or [BlackBoxOptim.jl](https://github.com/robertfeldt/BlackBoxOptim.jl) as its backend. Currently our recommendation is to use BlackBoxOptim and the `bbextract` function since it seems to be faster and better at finding the global maximum. Additionally if you want to use gradient based methods, we default to finite difference methods since Zygote.jl autodiff seems to be quite slow, although we may improve this in the future. 
If you use Optim.jl we found our solution got stuck in local minima even if using simulated annealing e.g. SAMIN()). To help with this you can pass use a threaded extraction where each thread runs an independent run started at different random locations. We found that this tends to find the global maximum if you use $>10$ initial guesses. 






## A minimal example of extracting ring features
We have provided a minimal example of how to run the filter in examples using command line arguments.
A simpler example is
```julia
  using VIDA
  #load the image and plot it
  image = load_ehtimfits("examples/data/elliptical_gaussian_rot-0.00.fits")
  plot(image)

  #Create the filter to use
  filter = TIDAGaussianRing(20.0, #20 μas Gaussian ring
                            5.0,  #std dev is 5.0 μas
                            0.2,  #1-b/a asymmetry b/a semi-minor/major
                            0.1,  #slash strength 0 is no slash 1 is max
                            π/4,  #ring orientation north of east
                            0.0,  #RA (x) location of ring center in μas
                            0.0   #DEC (y) locatin of ring center in μas
                           )
  #Plot the filter
  plot(filter)

  #make the measure you can choose from :KL or :Bh currently.
  bh = Bhattacharyya(image)
  
  #To call the function bh
  bh(filter)
          
  #parameter bounds
  lower = [5.0 0.1, 1e-3, -0.99, -π, -50, -50]
  upper = [40.0 30.0, 0.99, 0.99, π, 50, 50]
  #Now extract!
  #filtermax is the filter that maximizes the bm,
  #bm_max is its max value
  #converved & itr are some run info to see if the optimizer said it reached convergence
  filtermax,bh_max,converged,itr = bbextract(bh, filter, lower, upper)
  
  #plot the results
  triptic(img, filtermax)
```

### Distributed computing
In the examples folder we have a complete script that shows how to use VIDA on a cluster to extract image features from multiple images at the same time. It uses argparse to read in command line options and a file that contains a list of paths of images to run VIDA on. 
