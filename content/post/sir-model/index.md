---
title: SIR model with deSolve and ggplot2
author: Eugene
date: '2017-01-04'
slug: sir-model
categories:
  - R
  - Infectious Diseases
tags: []
subtitle: ''
summary: ''
lastmod: '2017-01-04'
projects: []
featured: false
image:
  caption: ""
  focal_point: ""
  preview_only: yes
---


Original post date: January 04 2017

In my infectious disease epidemiology course back in 2016, I first learned about a basic **compartmental disease transmission model**. I thought it was pretty awesome to predict and model how an infectious agent could affect a population. Purely out of interest I wanted to develop some working code in R to make a function that would plot the model based on specified parameters.  

The basic deterministic model is composed of three compartments that represent different categories of individuals within a population; the **susceptible**, **infected**, and **recovered** – hence the **SIR model**. The relationship of these three groups is described by a set of differential equations first derived by [Kermack and McKendrick](https://dx.doi.org/10.1098%2Frspa.1927.0118). The SIR model details the transmission of infection through the contact of susceptible individuals with an infected host. If you are interested in learning more on this model, there is an [online module](http://www.maa.org/press/periodicals/loci/joma/the-sir-model-for-spread-of-disease-the-differential-equation-model).

# The SIR Model

<center>
dS/dt = -βSI

dI/dt = βSI – γI

dR/dt = γI

β = cp
<br>
</center>
<br>

 *S* – proportion of susceptible individuals in total population  
 *I* – proportion of infected individuals in total population  
 *R* – proportion of recovered individuals in total population  
 *β* – transmission parameter (rate of infection for susceptible-infected contact)  
 *c* – number of contacts each host has per unit time (contact rate)  
 *p* – probability of transmission of infection per contact (transmissibility)  
 *γ* – recovery parameter (rate of infected transitioning to recovered)  

This model is very basic and has important assumptions. The first being the population is closed and fixed, in other words – no one it added into the susceptible group (no births), all individuals who transition from being infected to recovered are permanently resistant to infection and there are no deaths. Second, the population is homogenous (all individuals are the same) and only differ by their disease state. Third, infection and that individual’s “infectiveness” or ability to infect susceptible individuals, occurs simultaneously.

# Now on to using R!

I used the [deSolve package](https://cran.r-project.org/web/packages/deSolve/index.html) which was developed to solve the initial condition values of differential equations in R. The code to setup the SIR model was adapted from the MATLAB code from [Modeling Infectious Diseases in Humans and Animals](http://homepages.warwick.ac.uk/~masfz/ModelingInfectiousDiseases/Chapter2/Program_2.1/index.html) and an [online demo](http://archives.aidanfindlater.com/blog/2010/04/20/the-basic-sir-model-in-r/) in R.

To begin, I setup my R function to be dependent on the input of three parameters: time period (in days), and the values of β and γ. These are denoted by t,b,g within the function.

The initial conditions are set to have the proportion of the populationg being in the Susceptible group at >99.9% (1-1E-6 to be exact), the Infected group to be close to 0 (1E-6) and no one in the Recovered group. The SIR model is going to be plotted at 0.5 days for higher resolution.  


```r
library(deSolve)
SIR.model <- function(t, b, g) {
  init <- c(S = 1 - 1e-6, I = 1e-6, R = 0)
  parameters <- c(bet = b, gamm = g)
  time <- seq(0, t, by = t / (2 * length(1:t)))
}
```


Next we setup the differential equation (from above) so that we can run the ode function from the deSolve package correctly.


```r
eqn <- function(time, state, parameters) {
  with(as.list(c(state, parameters)), {
    dS <- -bet * S * I
    dI <- bet * S * I - gamm * I
    dR <- gamm * I
    return(list(c(dS, dI, dR)))
  })
}
```

Then we run the ode function based on the parameters we set above and save coerce the output as a data frame class.


```r
out <- ode(y=init, times=time, eqn, parms=parameters)
out.df <- as.data.frame(out)
```

Now that we’ve solved the differential equation, the next task is to plot it. To do this I am going to be using the elegant visualization package [ggplot2](http://ggplot2.org/) (version 2.1.0).

First I’m going to set a theme for the plot, by modifying one of the built-in ggplot2 themes, `theme_bw`. I referred to ggplot2 documentation found [here](http://docs.ggplot2.org/current/theme.html#).


```r
library(ggplot2)
mytheme4 <- theme_bw() +
  theme(text = element_text(colour = "black")) +
  theme(panel.grid = element_line(colour = "white")) +
  theme(panel.background = element_rect(fill = "#B2B2B2"))
  theme_set(mytheme4)
```

Here I set the title for the plot as SIR Model: Basic. For the subtitle it will show the values of β and γ set when first running the function. The use of `bquote` was based reading some [material](http://adv-r.had.co.nz/Expressions.html) on R expressions, trial and error, with some scouring of [Stack Overflow](http://stackoverflow.com/questions/tagged/r) (which is an incredible resource FYI).


```r
title <- bquote("SIR Model: Basic")
subtit <- bquote(list(beta==.(parameters[1]),~gamma==.(parameters[2])))
```

Now I describe the plot, essentially I am selecting the data frame from the `ode` function. Most of the code for `theme` and `legend` is just for aesthetics.


```r
res <- ggplot(out.df, aes(x = time)) +
  ggtitle(bquote(atop(bold(.(
    title
  )), atop(bold(
    .(subtit)
  ))))) +
  geom_line(aes(y = S, colour = "Susceptible")) +
  geom_line(aes(y = I, colour = "Infected")) +
  geom_line(aes(y = R, colour = "Recovered")) +
  ylab(label = "Proportion") +
  xlab(label = "Time (days)") +
  theme(legend.justification = c(1, 0),
        legend.position = c(1, 0.5)) +
  theme(
    legend.title = element_text(size = 12, face = "bold"),
    legend.background = element_rect(
      fill = '#FFFFFF',
      size = 0.5,
      linetype = "solid"
    ),
    legend.text = element_text(size = 10),
    legend.key = element_rect(
      colour = "#FFFFFF",
      fill = '#C2C2C2',
      size = 0.25,
      linetype = "solid"
    )
  ) +
  scale_colour_manual(
    "Compartments",
    breaks = c("Susceptible", "Infected", "Recovered"),
    values = c("blue", "red", "darkgreen")
  )
```

So now the function runs, it prints the plot and write a .png image file to the file directory with the specified parameters in the filename. The ggsave function accomplishes this with ease.


```r
print(res)
ggsave(
  plot=res, 
  filename=paste0("SIRplot_","time",t,"beta",b,"gamma",g,".png"), 
  width=8,
  height=6,
  dpi=180)
```

The fully annotated code can be found in my GitHub repository SIR-interact. You can download the code and tinker around with it.

So when we run the function while setting the period to 70 days, β = 1.4 and γ = 0.3 `SIR.model(70,1.4,0.3)` we write a file with the name `SIRplot_time70beta1.4gamma0.3.png` and get output plot looking like…

![](/post/sir-model-with-desolve-and-ggplot2/2017-01-04-sir-model-with-desolve-and-ggplot2_files/sir_plot.png){width=95%}

