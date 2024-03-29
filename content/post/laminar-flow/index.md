---
title: Laminar flow with ggplot2 and gganimate
author: Eugene
date: '2018-04-05'
slug: laminar-flow
categories:
  - R
  - Hemodynamics
tags: []
subtitle: ''
summary: ''
projects: []
featured: false
image:
  caption: ""
  focal_point: ""
  preview_only: yes
---


Original post date: April 05 2018

This post isn't public health related (like this one) but they will be more of a personal exploration and application of different tools and packages in R. This post will be exploring the animation and gganimate package to create some (hopefully) cool animated visualizations.

## Laminar Flow
A simple definition of [laminar flow](https://en.wikipedia.org/wiki/Laminar_flow) is when a fluid flows through a pipe in parallel layers with no disruptions between these layers. Turbulent flow occurs when there is mixing or any disruption between these layers.  Laminar flow is a simple analysis of fluid dynamics that considers the viscosity (“thickness/gooeyness”) of a liquid. This model is based on a number of assumptions where the fluid flows through a straight cylindrical pipe of fixed diameter, there is no acceleration (only a balance between pressure and viscous/shear forces) and gravitational forces are ignored. The walls of the pipe exhibit the maximum [shearing force](https://en.wikipedia.org/wiki/Shear_force) and the axial centre of the pipe has zero shear. This results in a parabolic [velocity profile](http://hyperphysics.phy-astr.gsu.edu/hbase/pfric2.html) where the velocity is at it’s maximum at centre of the pipe.

## MATLAB to R?
First you might wonder, why is this post randomly on laminar flow? A short trip down memory lane while organizing some undergraduate course folders resulted in my rediscovery of some lecture notes and code related to medical biophysics and hemodynamics. Not having used MATLAB in over 4 years, **I was curious whether I could translate some of the MATLAB code into something similar into R**. Some points on the [differences](https://www.mathworks.com/discovery/matlab-vs-r.html), MATLAB is a proprietary numerical computing programming software based on matrix algorithms while R is a open-source programming language centered on statistical analyses. There are some StackOverflow [discussions](https://stackoverflow.com/questions/1738087/what-can-matlab-do-that-r-cannot-do) on these comparisons if you want to explore this further. I also discovered the `pracma` [package](https://cran.r-project.org/web/packages/pracma/index.html) for numerical analysis that uses some MATLAB function names (#tbt). Now to the actual R code!


## Initial Conditions
First we’ll load the relevant packages. I’ve commented in the code lines on some basic descriptions on each package. Just a small note that some of these packages need to be downloaded from GitHub repositories using the `devtools` [package](https://cran.r-project.org/web/packages/devtools/index.html).


```r
# Packages ####
library(pracma) #load practical math package
library(ggplot2) #data viz
library(scales) #plot scales
library(dplyr) #data wrangling
library(viridis) #data viz palette
library(animation) #animation
library(gganimate) #ggplot animation
```

Next, for any type of model you need to first set the **initial conditions** or parameters. In this case, we’ll specify the conditions for the pressure difference at the beginning at the end of the pipe, the viscous force, the dimensions of the pipe (radius and length).

```r
## Initial Conditions ####
# Pressure difference (delta P)
Pi <- 10 #initial pressure [Pa]
Pf <- 50 #final pressure [Pa]
 
## Boundary Conditions ####
# Pipe and fluid characteristics
mu <- 8.94e-4 #viscosity for water. units: [Pa*s] at temp = 25 C
R <- 0.1 #radius of pipe [m]
l <- 4 #length of pipe [m]
 
N <- 100 #radial resolution
r <- seq(-R,R,length.out=N) #diameter of pipe
S <- pi*(R^2) #cross-sectional area of pipe
```

Since we're also playing with the `animation` [package](https://cran.r-project.org/web/packages/animation/index.html), we’ll also input a vector of time units to loop and animate the visualization.


```r
# time dependent
time <- seq(0,19) #length 20
Pit <- 0.2*time*Pi
Pft <- 0.2*time*Pf
Pdiff <- abs(Pft-Pit) #pressure difference over time
```

## Data Setup
Now what we’ll do is run a loop over the time sequence we’ve specified (20 arbitrary units of time, seconds for example) to create the output data. The pressure difference will be increased as a function of time, simulating increased flow of the fluid (water in this case) over time. The equation in the code below is the flow velocity as a function of the pipe radius derived from [Hagen-Poiseuille’s Law](https://en.wikipedia.org/wiki/Hagen%E2%80%93Poiseuille_equation) specified below. So after running `lapply()` over the pressure differences, we then massage the list output into a data frame – first by “unlisting” the list elements, creating a matrix object where each column represents a different time point, and coercing the matrix into a data frame (and as always [remembering](https://simplystatistics.org/2015/07/24/stringsasfactors-an-unauthorized-biography/) to use `stringsAsFactors = FALSE`).

> Hagen-Poiseuille’s Law  
> Q = πR4Δp / 8μl


```r
# Create Data Viz ####
# plot shows V (average speed) and r (pipe radius)
# ((Pdiff)*(R^2)/(4*mu*l))*(1-(r^2)/R^2) #eqn for V
out1 <- lapply(seq_along(Pdiff), function(x) {
  ((Pdiff[x]) * (R ^ 2) / (4 * mu * l)) * (1 - (r ^ 2) / R ^ 2)
})

out1_df <-data.frame(
  matrix(unlist(out1), ncol = length(time)), 
  stringsAsFactors = FALSE
  )

names(out1_df) <- seq_along(names(out1_df))
```

Now we have a data frame with each column containing the velocity of the fluid as a function of the radius. There are twenty time periods and 100 rows representing the resolution of the x-axis (e.g. 100 ticks). The problem now is that this data frame is not in a `ggplot2` friendly format. We’ll have to use the `tidyr` [package](https://cran.r-project.org/web/packages/tidyr/index.html) to manipulate the data. The code below ["gathers"](http://garrettgman.github.io/tidying/) the columns so they become grouped elements in one column and then we use the `dplyr` [package](https://cran.r-project.org/web/packages/dplyr/index.html) to add the radius values as new column, coerce the "time" column into an integer type, and add the flowrate (a function of the velocity profile and pipe cross-sectional area) as another column. A note here that when the `rbind()` function is used, if the length/# of rows of the object’s (`r`) being "binded"" is a multiple of the other object’s (`df_t`) length/# of rows, it will repeat based on the multiple. Since we looped based on the the radial resolution (100) specified before, `rbind()` will fill the column with 20 multiples (20*100) of the radius ticks.


```r
df_t <- tidyr::gather(out1_df,key="time",value="V")
 
df_t2 <- df_t %>% #previous object
mutate(time = as.integer(time)) %>% #change to integer type
  mutate(Qv = V/(1-(r^2)/R^2)) #add volumetric flowrate (Qv)
```

## Data Visualization
Now we have our data in a nice format to visualize and loop over to create our GIF. We specify the data frame we want to use, map the aesthetics and specifically the `frame` and `cumulative` arguments. The `frame` aesthetic tells the `gganimate` [package](https://github.com/dgrtwo/gganimate) to loop over the values defined in `frame` or time in our case. We’ll use the amazing `viridis` package to add some time colours, use the built-in `ggplot2` `theme_dark`, tinker with the text sizes, and provide a good title and axis labels (yay for plotting best practices). The legend here displays the flow rate through the pipe going from purple (low flowrate) to yellow (high flowrate).


```r
p1 <- ggplot(data=df_t2, aes(y=r,x=V,col=Qv,frame=time,cumulative=TRUE)) +
geom_path() + geom_point(alpha=0.05) +
scale_colour_viridis(option = "C", discrete = FALSE) +
theme_dark() +
labs(title = "Hagen-Poiseuille's Law: Velocity Profile t =",
y = "Pipe Radius (m)", x = "Velocity (m/s)")
 
ggsave(p1,filename = "Vprofile1.png",width = 250, height=100,units = "mm")
```

![](/post/laminar-flow/vprofile1.png){width=99%}

The title might look strange but the empty "t = "" will be useful to indicate which time point each frame describes when we create the .gif (look at the `title_frame` argument in the `gganimate()` function below. This will add the value provided to the frame aesthetic and include it into the plot title. We also specify the intervals for each frame (0.2 seconds) and the dimensions of the output.


```r
gganimate(p1,filename = "Vprofile1.gif", title_frame = TRUE,
interval=0.2, ani.width = 900, ani.height = 350)
```

![](/post/laminar-flow/vprofile11.gif){width=105%}

Now we have this nice animation visualizing the laminar flow velocity profile of a pipe at different time points where the pressure difference increases, in turn increasing the maximum velocity of the fluid. Another way to see it, is that at time 0, there is no pressure difference, therefore no flow. But once the pressure difference increases (decreased lower pressure at the end of pipe e.g. increased flow of fluid out of the end of the pipe) the velocity profile or flow increases.

The entire script containing the code is found in my GitHub repo if you would want to fork it or tinker with it yourself.
