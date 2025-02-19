---
title: "R regression Bayesian (using brms)"
author: "By [Laurent Smeets](https://www.rensvandeschoot.com/colleagues/laurent-smeets/) and [Rens van de Schoot](https://www.rensvandeschoot.com/about-rens/)"
date: 'Last modified: 22 August 2019'
output:
  html_document:
    keep_md: true
---

This tutorial provides the reader with a basic tutorial how to perform a **Bayesian regression** in [brms](https://cran.r-project.org/web/packages/brms/index.html),  using Stan instead of  as the MCMC sampler. Throughout this tutorial, the reader will be guided through importing data files, exploring summary statistics and regression analyses. Here, we will exclusively focus on [Bayesian](https://www.rensvandeschoot.com/a-gentle-introduction-to-bayesian-analysis-applications-to-developmental-research/) statistics. 

In this tutorial, we start by using the default prior settings of the software.  In a second step, we will apply user-specified priors, and if you really want to use Bayes for your own data, we recommend to follow the [WAMBS-checklist](https://www.rensvandeschoot.com/wambs-checklist/), also available in other software.

## Preparation

This tutorial expects:

- Installation of [STAN](https://mc-stan.org/users/interfaces/rstan) and [Rtools](https://cran.r-project.org/bin/windows/Rtools). For more information please see https://github.com/stan-dev/rstan/wiki/RStan-Getting-Started
- Installation of R packages `rstan`, and `brms`. This tutorial was made using brms version 2.9.0 in R version 3.6.1
- Basic knowledge of hypothesis testing
- Basic knowledge of correlation and regression
- Basic knowledge of [Bayesian](https://www.rensvandeschoot.com/a-gentle-introduction-to-bayesian-analysis-applications-to-developmental-research/) inference
- Basic knowledge of coding in R


## Example Data

The data we will be using for this exercise is based on a study about predicting PhD-delays ([Van de Schoot, Yerkes, Mouw and Sonneveld 2013](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0068839)).The data can be downloaded [here](https://www.rensvandeschoot.com/wp-content/uploads/2018/10/phd-delays.csv). Among many other questions, the researchers asked the Ph.D. recipients how long it took them to finish their Ph.D. thesis (n=333). It appeared that Ph.D. recipients took an average of 59.8 months (five years and four months) to complete their Ph.D. trajectory. The variable B3_difference_extra measures the difference between planned and actual project time in months (mean=9.97, minimum=-31, maximum=91, sd=14.43). For more information on the sample, instruments, methodology and research context we refer the interested reader to the paper.

For the current exercise we are interested in the question whether age (M = 31.7, SD = 6.86) of the Ph.D. recipients is related to a delay in their project.

The relation between completion time and age is expected to be non-linear. This might be due to that at a certain point in your life (i.e., mid thirties), family life takes up more of your time than when you are in your twenties or when you are older.

So, in our model the $gap$ (*B3_difference_extra*) is the dependent variable and $age$ (*E22_Age*) and $age^2$(*E22_Age_Squared *) are the predictors. The data can be found in the file <span style="color:red"> ` phd-delays.csv` </span>.

  <p>&nbsp;</p>


##### _**Question:** Write down the null and alternative hypotheses that represent this question. Which hypothesis do you deem more likely?_

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]

$H_0:$ _$age$ is not related to a delay in the PhD projects._

$H_1:$ _$age$ is related to a delay in the PhD projects._ 

$H_0:$ _$age^2$ is not related to a delay in the PhD projects._

$H_1:$ _$age^2$is related to a delay in the PhD projects._ 

[/expand]


  <p>&nbsp;</p>

## Preparation - Importing and Exploring Data



```r
# if you dont have these packages installed yet, please use the install.packages("package_name") command.
library(rstan) 
library(brms)
library(psych) #to get some extended summary statistics
library(tidyverse) # needed for data manipulation and plotting
```

You can find the data in the file <span style="color:red"> ` phd-delays.csv` </span>, which contains all variables that you need for this analysis. Although it is a .csv-file, you can directly load it into R using the following syntax:

```r
#read in data
dataPHD <- read.csv2(file="phd-delays.csv")
colnames(dataPHD) <- c("diff", "child", "sex","age","age2")
```


Alternatively, you can directly download them from GitHub into your R work space using the following command:

```r
dataPHD <- read.csv2(file="https://raw.githubusercontent.com/LaurentSmeets/Tutorials/master/Blavaan/phd-delays.csv")
colnames(dataPHD) <- c("diff", "child", "sex","age","age2")
```

GitHub is a platform that allows researchers and developers to share code, software and research and to collaborate on projects (see https://github.com/)

Once you loaded in your data, it is advisable to check whether your data import worked well. Therefore, first have a look at the summary statistics of your data. you can do this by using the  `describe()` function.

##### _**Question:** Have all your data been loaded in correctly? That is, do all data points substantively make sense? If you are unsure, go back to the .csv-file to inspect the raw data._

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
describe(dataPHD)
```

```
##       vars   n    mean     sd median trimmed    mad min  max range  skew
## diff     1 333    9.97  14.43      5    6.91   7.41 -31   91   122  2.21
## child    2 333    0.18   0.38      0    0.10   0.00   0    1     1  1.66
## sex      3 333    0.52   0.50      1    0.52   0.00   0    1     1 -0.08
## age      4 333   31.68   6.86     30   30.39   2.97  26   80    54  4.45
## age2     5 333 1050.22 656.39    900  928.29 171.98 676 6400  5724  6.03
##       kurtosis    se
## diff      5.92  0.79
## child     0.75  0.02
## sex      -2.00  0.03
## age      24.99  0.38
## age2     42.21 35.97
```

_The descriptive statistics make sense:_

_diff: Mean (9.97), SE (0.791)_

_$Age$: Mean (31.68), SE (0.38)_

_$Age^2$: Mean (1050.22), SE (35.97)_

[/expand]

  <p>&nbsp;</p>

## plot

Before we continue with analyzing the data we can also plot the expected relationship.


```r
dataPHD %>%
  ggplot(aes(x = age,
             y = diff)) +
  geom_point(position = "jitter",
             alpha    = .6)+ #to add some random noise for plotting purposes
  theme_minimal()+
  geom_smooth(method = "lm",  # to add  the linear relationship
              aes(color = "linear"),
              se = FALSE) +
  geom_smooth(method = "lm",
              formula = y ~ x + I(x^2),# to add  the quadratic relationship
              aes(color = "quadratic"),
              se = FALSE) +
  labs(title    = "Delay vs. age",
       subtitle = "There seems to be some quadratic relationship",
       x        = "Age",
       y        = "Delay",
       color    = "Type of relationship" ) +
  theme(legend.position = "bottom")
```

![](R-regression-bayesian--using-brms-_files/figure-html/unnamed-chunk-5-1.png)<!-- -->


## Regression - Default Priors

In this exercise you will investigate the impact of Ph.D. students&#39; $age$ and $age^2$ on the delay in their project time, which serves as the outcome variable using a regression analysis (note that we ignore assumption checking!).



As you know, Bayesian inference consists of combining a prior distribution with the likelihood obtained from the data. Specifying a prior distribution is one of the most crucial points in Bayesian inference and should be treated with your highest attention (for a quick refresher see e.g.  [Van de Schoot et al. 2017](http://onlinelibrary.wiley.com/doi/10.1111/cdev.12169/abstract)). In this tutorial, we will first rely on the default prior settings, thereby behaving a &#39;naive&#39; Bayesians (which might [not](https://www.rensvandeschoot.com/analyzing-small-data-sets-using-bayesian-estimation-the-case-of-posttraumatic-stress-symptoms-following-mechanical-ventilation-in-burn-survivors/) always be a good idea).

To run a multiple regression with brms, you first specify the model, then fit the model and finally acquire the summary (similar to the frequentist model using  `lm()`). The model is specified as follows:


1.  A dependent variable we want to predict.
2.  A "~", that we use to indicate that we now give the other variables of interest.
    (comparable to the '=' of the regression equation).
3.  The different independent variables separated by the summation symbol '+'.
4.  Finally, we insert that the dependent variable has a variance and that we
    want an intercept.  
5. We do set a seed to make the results exactly reproducible. 

There are many other options we can select, such as the number of chains how many iterations we want and how long of a warm-up phase we want, but we will just use the defaults for now.

For more information on the basics of brms, see the [website and vignettes](https://cran.r-project.org/web/packages/brms/index.html) 


  <p>&nbsp;</p>

The following code is how to specify the regression model:

```r
# 1) specify the model
model <- brm(formula = diff ~ age + age2, 
             data    = dataPHD,
             seed    = 123)
```


Now we will have a look at the summary by using `summary(model)` or  `posterior_summary(model)` for more precise estimates of the coefficients 


[expand title="Show Output" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
summary(model)
```

```
##  Family: gaussian 
##   Links: mu = identity; sigma = identity 
## Formula: diff ~ age + age2 
##    Data: dataPHD (Number of observations: 333) 
## Samples: 4 chains, each with iter = 2000; warmup = 1000; thin = 1;
##          total post-warmup samples = 4000
## 
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## Intercept   -47.31     12.66   -71.99   -23.49       2068 1.00
## age           2.66      0.60     1.53     3.84       2030 1.00
## age2         -0.03      0.01    -0.04    -0.01       2038 1.00
## 
## Family Specific Parameters: 
##       Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## sigma    14.05      0.56    13.01    15.19       2914 1.00
## 
## Samples were drawn using sampling(NUTS). For each parameter, Eff.Sample 
## is a crude measure of effective sample size, and Rhat is the potential 
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

```r
posterior_summary(model)
```

```
##                  Estimate    Est.Error          Q2.5         Q97.5
## b_Intercept -4.731198e+01 12.657267470 -7.199338e+01 -2.348821e+01
## b_age        2.664366e+00  0.599458026  1.534526e+00  3.836621e+00
## b_age2      -2.587054e-02  0.006261891 -3.825559e-02 -1.408062e-02
## sigma        1.404553e+01  0.560142134  1.301342e+01  1.518882e+01
## lp__        -1.356619e+03  1.431719503 -1.360214e+03 -1.354822e+03
```

[/expand]


  <p>&nbsp;</p> 
  
  
  The results that stem from a Bayesian analysis are genuinely different from those that are provided by a frequentist model. 

[expand title=Read more on the Bayesian analysis]

The key difference between Bayesian statistical inference and frequentist statistical methods concerns the nature of the unknown parameters that you are trying to estimate. In the frequentist framework, a parameter of interest is assumed to be unknown, but fixed. That is, it is assumed that in the population there is only one true population parameter, for example, one true mean or one true regression coefficient. In the Bayesian view of subjective probability, all unknown parameters are treated as uncertain and therefore are be described by a probability distribution. Every parameter is unknown, and everything unknown receives a distribution.


This is why in frequentist inference, you are primarily provided with a point estimate of the unknown but fixed population parameter. This is the parameter value that, given the data, is most likely in the population. An accompanying confidence interval tries to give you further insight in the uncertainty that is attached to this estimate. It is important to realize that a confidence interval simply constitutes a simulation quantity. Over an infinite number of samples taken from the population, the procedure to construct a (95%) confidence interval will let it contain the true population value 95% of the time. This does not provide you with any information how probable it is that the population parameter lies within the confidence interval boundaries that you observe in your very specific and sole sample that you are analyzing. 

In Bayesian analyses, the key to your inference is the parameter of interest&#39;s posterior distribution. It fulfills every property of a probability distribution and quantifies how probable it is for the population parameter to lie in certain regions. On the one hand, you can characterize the posterior by its mode. This is the parameter value that, given the data and its prior probability, is most probable in the population. Alternatively, you can use the posterior&#39;s mean or median. Using the same distribution, you can construct a 95% credibility interval, the counterpart to the confidence interval in frequentist statistics. Other than the confidence interval, the Bayesian counterpart directly quantifies the probability that the population value lies within certain limits. There is a 95% probability that the parameter value of interest lies within the boundaries of the 95% credibility interval. Unlike the confidence interval, this is not merely a simulation quantity, but a concise and intuitive probability statement. For more on how to interpret Bayesian analysis, check [Van de Schoot et al. 2014](http://onlinelibrary.wiley.com/doi/10.1111/cdev.12169/abstract).

[/expand]

  <p>&nbsp;</p> 

##### _**Question:** Interpret the estimated effect, its interval and the posterior distribution._

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]

_$Age$ seems to be a relevant predictor of PhD delays, with a posterior mean regression coefficient of 2.67, 95% Credibility Interval [1.53,  3.83]. Also, $age^2$ seems to be a relevant predictor of PhD delays, with a posterior mean of  -0.0259, and a 95% credibility Interval of [-0.038, -0.014]. The 95% Credibility Interval shows that there is a 95% probability that these regression coefficients in the population lie within the corresponding intervals, see also the posterior distributions in the figures below. Since 0 is not contained in the Credibility Interval we can be fairly sure there is an effect._



[/expand]

  <p>&nbsp;</p> 

##### _**Question:** Every Bayesian model uses a prior distribution. Describe the shape of the prior distributions of the regression coefficients._

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]


To check which default priors are being used by brms, you can use the `prior_summary()` function or check the brms [documentation](https://cran.r-project.org/web/packages/brms/brms.pdf), which states that, _"The default prior for population-level effects (including monotonic and category specific effects) is an improper flat prior over the reals"_ This means, that there an uninformative prior was chosen. 


```r
prior_summary(model)
```

```
##                 prior     class coef group resp dpar nlpar bound
## 1                             b                                 
## 2                             b  age                            
## 3                             b age2                            
## 4 student_t(3, 5, 10) Intercept                                 
## 5 student_t(3, 0, 10)     sigma
```

We also see that a student-t distribution was chosen for the intercept.

[/expand]

  <p>&nbsp;</p> 


## **Regression - User-specified Priors**


In brms, you can also manually specify your prior distributions. Be aware that usually, this has to be done BEFORE peeking at the data, otherwise you are double-dipping (!). In theory, you can specify your prior knowledge using any kind of distribution you like. However, if your prior distribution does not follow the same parametric form as your likelihood, calculating the model can be computationally intense. _Conjugate_ priors avoid this issue, as they take on a functional form that is suitable for the model that you are constructing. For your normal linear regression model, conjugacy is reached if the priors for your regression parameters are specified using normal distributions (the residual variance receives an inverse gamma distribution, which is neglected here). In brms, you are quite flexible in the specification of informative priors.

Let&#39;s re-specify the regression model of the exercise above, using conjugate priors. We leave the priors for the intercept and the residual variance untouched for the moment. Regarding your regression parameters, you need to specify the hyperparameters of their normal distribution, which are the mean and the variance. The mean indicates which parameter value you deem most likely. The variance expresses how certain you are about that. We try 4 different prior specifications, for both the $\beta_{age}$ regression coefficient, and the $\beta_{age^2}$ coefficient.

First, we use the following prior specifications:

_$Age$ ~  $\mathcal{N}(3, 0.4)$_

_$Age^2$ ~  $\mathcal{N}(0, 0.1)$_


In brms, the priors are set using the `set_prior()` function. Be careful, Stan uses standard deviations instead of variance in the normal distribution. The standard deviations is the square root of the variance, so a variance of 0.1 corresponds to a standard deviation of 0.316 and a variance of 0.4 corresponds to a standard deviation of 0.632.

The priors are presented in code as follows:


```r
priors2 <- c(set_prior("normal(3, 0.632)", class = "b", coef = "age"),
             set_prior("normal(0, 0.316)", class = "b", coef = "age2"))
```

Now we can run the model again, but with the `prior=` included. Now fit the model again and request for summary statistics. 

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]



```r
model2 <- brm(formula = diff ~  age + age2, 
              data    = dataPHD,
              prior   = priors2,
              seed    = 123)
```


```r
summary(model2)
```

```
##  Family: gaussian 
##   Links: mu = identity; sigma = identity 
## Formula: diff ~ age + age2 
##    Data: dataPHD (Number of observations: 333) 
## Samples: 4 chains, each with iter = 2000; warmup = 1000; thin = 1;
##          total post-warmup samples = 4000
## 
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## Intercept   -50.14      9.45   -68.37   -30.86       1733 1.00
## age           2.80      0.45     1.90     3.70       1658 1.00
## age2         -0.03      0.00    -0.04    -0.02       1698 1.00
## 
## Family Specific Parameters: 
##       Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
## sigma    14.02      0.54    13.03    15.09       2550 1.00
## 
## Samples were drawn using sampling(NUTS). For each parameter, Eff.Sample 
## is a crude measure of effective sample size, and Rhat is the potential 
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

```r
posterior_summary(model2)
```

```
##                  Estimate   Est.Error          Q2.5         Q97.5
## b_Intercept -5.013647e+01 9.454018945 -6.837267e+01 -3.086225e+01
## b_age        2.801673e+00 0.448175862  1.898214e+00  3.697464e+00
## b_age2      -2.730237e-02 0.004752593 -3.684538e-02 -1.784013e-02
## sigma        1.401859e+01 0.537796489  1.302873e+01  1.509395e+01
## lp__        -1.356921e+03 1.425728330 -1.360569e+03 -1.355141e+03
```

[/expand]




#### _**Question** Fill in the results in the table below:_


| $Age$          | Default prior              |$\mathcal{N}(3, .4)$| $\mathcal{N}(3, 1000)$|$\mathcal{N}(20, .4)$|$\mathcal{N}(20, 1000)$|
| ---            | ---                        | ---                    | ---        |---         | ---         |
| Posterior mean |  2.664|                        |            |            |             | 
| Posterior sd   |  0.599|                        |            |            |             |

| $Age^2$        | Default prior             | $\mathcal{N}(0, .1)$   |  $\mathcal{N}(0, 1000)$ |  $\mathcal{N}(20, .1)$|  $\mathcal{N}(20, 1000)$ |
| ---            | ---                       | ---       | ---        | ---        | ---         |
| Posterior mean | -0.026|           |            |            |             |
| Posterior sd   | 0.006|           |            |            |             |



[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]






| $Age$          | Default prior              |$\mathcal{N}(3, .4)$| $\mathcal{N}(3, 1000)$|$\mathcal{N}(20, .4)$|$\mathcal{N}(20, 1000)$|
| ---            | ---                        | ---                    | ---        |---         | ---         |
| Posterior mean |  2.664|2.802|            |            |             | 
| Posterior sd   |  0.599|0.448            |            |             |

| $Age^2$        | Default prior             | $\mathcal{N}(0, .1)$   |  $\mathcal{N}(0, 1000)$ |  $\mathcal{N}(20, .1)$|  $\mathcal{N}(20, 1000)$ |
| ---            | ---                       | ---       | ---        | ---        | ---         |
| Posterior mean | -0.026|-0.027|            |            |             |
| Posterior sd   | 0.006|0.005 |            |            |             |

[/expand]

Next, try to adapt the code, using the prior specifications of the other columns and then complete the table.


[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
priors3 <- c(set_prior("normal(3, 31.6)", class = "b", coef = "age"),
             set_prior("normal(0, 31.6)", class = "b", coef = "age2"))

model3 <- brm(formula = diff ~  age + age2, 
              data    = dataPHD,
              prior   = priors3,
              seed    = 123)


priors4 <- c(set_prior("normal(20,0.632)", class = "b", coef = "age"),
             set_prior("normal(20,0.316)", class = "b", coef = "age2"))

model4 <- brm(formula = diff ~  age + age2, 
              data    = dataPHD,
              prior   = priors4,
              seed    = 123)


priors5 <- c(set_prior("normal(20,31.6)", class = "b", coef = "age"),
             set_prior("normal(20,31.6)", class = "b", coef = "age2"))

model5 <- brm(formula = diff ~  age + age2, 
              data    = dataPHD,
              prior   = priors5,
              seed    = 123)
```


```r
posterior_summary(model3)
```

```
##                  Estimate    Est.Error          Q2.5         Q97.5
## b_Intercept -4.706608e+01 12.429187704   -71.1808151 -2.330294e+01
## b_age        2.653895e+00  0.589350223     1.5157321  3.803406e+00
## b_age2      -2.577423e-02  0.006138373    -0.0377718 -1.393926e-02
## sigma        1.402546e+01  0.553730999    12.9905373  1.511654e+01
## lp__        -1.365352e+03  1.449436172 -1369.0190205 -1.363574e+03
```

```r
posterior_summary(model4)
```

```
##                  Estimate    Est.Error          Q2.5         Q97.5
## b_Intercept  -262.9455720 13.234422259  -289.4484994  -238.1084417
## b_age          12.9512563  0.621010157    11.7771003    14.1744210
## b_age2         -0.1308252  0.006523318    -0.1435894    -0.1184722
## sigma          19.4938823  0.936356787    17.7521854    21.3993495
## lp__        -3558.4549291  1.433473206 -3562.0006962 -3556.6814001
```

```r
posterior_summary(model5)
```

```
##                  Estimate    Est.Error          Q2.5         Q97.5
## b_Intercept -4.709795e+01 12.600564696 -7.206517e+01 -2.288395e+01
## b_age        2.655475e+00  0.596027523  1.494316e+00  3.845159e+00
## b_age2      -2.579004e-02  0.006191026 -3.822255e-02 -1.384521e-02
## sigma        1.403870e+01  0.538621642  1.303185e+01  1.513337e+01
## lp__        -1.365711e+03  1.410203900 -1.369248e+03 -1.363932e+03
```





  
| $Age$          | Default prior              |$\mathcal{N}(3, .4)$| $\mathcal{N}(3, 1000)$|$\mathcal{N}(20, .4)$|$\mathcal{N}(20, 1000)$|
| ---            | ---                        | ---                    | ---        |---         | ---         |
| Posterior mean |  2.664|2.802  |2.654 |12.951|2.655| 
| Posterior sd   |  0.599|0.448  |0.589|0.621|0.596| 

| $Age^2$        | Default prior             | $\mathcal{N}(0, .1)$   |  $\mathcal{N}(0, 1000)$ |  $\mathcal{N}(20, .1)$|  $\mathcal{N}(20, 1000)$ |
| ---            | ---                       | ---       | ---        | ---        | ---         |
| Posterior mean | -0.026|-0.027 |-0.026|-0.131|-0.026|
| Posterior sd   | 0.006|0.005 |0.006|0.007|0.006|


[/expand]

#### _**Question**: Compare the results over the different prior specifications. Are the results comparable with the default model?_

#### _**Question**: Do we end up with similar conclusions, using different prior specifications?_

_To answer these questions, proceed as follows: We can calculate the relative bias to express this difference. ($bias= 100*\frac{(model \; informative\; priors\;-\;model \; uninformative\; priors)}{model \;uninformative \;priors}$). In order to preserve clarity we will just calculate the bias of the two regression coefficients and only compare the default (uninformative) model with the model that uses the $\mathcal{N}(20, .4)$ and $\mathcal{N}(20, .1)$ priors. Copy Paste the following code to R:_



```r
#1) get the estimates
estimatesuninformative <- posterior_summary(model)[c("b_age", "b_age2") , "Estimate"]
estimatesinformative   <- posterior_summary(model4)[c("b_age", "b_age2") , "Estimate"]

#2) calculate the bias
round(100*((estimatesinformative-estimatesuninformative)/estimatesuninformative), 2)
```

The `b_age` and `b_age2` indices stand for the $\beta_{age}$ and $\beta_{age^2}$ respectively. 


We can also plot these differences by plotting both the posterior and priors for the five different models we ran. In this example we only plot the regression of coefficient of age $\beta_{age}$

First we extract the MCMC chains of the 5 different models for only this one parameter ($\beta_{age}$=beta[1,2,1]). Copy-past the following code to R: 

[expand title="Show Syntax" trigclass="noarrow my_button" targclass="my_content" tag="button"]



```r
posterior1.2.3.4.5 <- bind_rows("uninformative prior" = as_tibble(as.mcmc(model,  pars = "b_age", exact_match = TRUE ,combine_chains = TRUE)),
                                "informative prior 1" = as_tibble(as.mcmc(model2, pars = "b_age", exact_match = TRUE ,combine_chains = TRUE)),
                                "informative prior 2" = as_tibble(as.mcmc(model3, pars = "b_age", exact_match = TRUE ,combine_chains = TRUE)),
                                "informative prior 3" = as_tibble(as.mcmc(model4, pars = "b_age", exact_match = TRUE ,combine_chains = TRUE)),
                                "informative prior 4" = as_tibble(as.mcmc(model5, pars = "b_age", exact_match = TRUE ,combine_chains = TRUE)),
                                .id = "id1")

prior1.2.3.4.5 <- bind_rows("uninformative prior" = enframe(rnorm(10000, mean=0, sd=sqrt(1/1e-2))),
                            "informative prior 1" = enframe(rnorm(10000, mean=3, sd=sqrt(0.4))),
                            "informative prior 2" = enframe(rnorm(10000, mean=3, sd=sqrt(1000))),
                            "informative prior 3" = enframe(rnorm(10000, mean=20, sd=sqrt(0.4))),
                            "informative prior 4" = enframe(rnorm(10000, mean=20, sd=sqrt(1000))),
                            .id = "id1") %>%
  rename(b_age = value)# here we sample a large number of values from the prior distributions to be able to plot them.

priors.posterior <- bind_rows("posterior" = posterior1.2.3.4.5, "prior" = prior1.2.3.4.5, .id = "id2")
```

instead of sampling the priors like this, you could also get the actual prior values sampled by Stan by adding the `  sample_prior = TRUE` command to the `brm()` function, this would save the priors as used by stan. With enough samples this would yield the same results


[/expand]

Then, we can plot the different posteriors and priors by using the following code:

[expand title="Show Syntax" trigclass="noarrow my_button" targclass="my_content" tag="button"]


```r
ggplot(data    = priors.posterior, 
       mapping = aes(x        = b_age, 
                     fill     = id1, 
                     colour   = id2, 
                     linetype = id2,
                     alpha    = id2))+
  geom_density(size=1)+
  scale_x_continuous(limits=c(0, 23))+
  scale_colour_manual(name   = 'Posterior/Prior', 
                      values = c("black","red"), 
                      labels = c("posterior", "prior"))+
  scale_linetype_manual(name   = 'Posterior/Prior',
                        values = c("solid","dotted"),
                        labels = c("posterior", "prior"))+
  scale_alpha_discrete(name   ='Posterior/Prior',
                       range  = c(.7,.3),
                       labels = c("posterior", "prior"))+
  scale_fill_manual(name   = "Densities",
                    values = c("Yellow","darkred","blue", "green", "pink"))+
  labs(title    = expression("Influence of (Informative) Priors on" ~ beta[Age]), 
       subtitle = "5 different densities of priors and posteriors")
```

![](R-regression-bayesian--using-brms-_files/figure-html/unnamed-chunk-19-1.png)<!-- -->
[/expand]

Now, with the information from the table, the bias estimates and the plot you can answer the two questions about the influence of the priors on the results. 

[expand title="Answer" trigclass="noarrow my_button" targclass="my_content" tag="button"]



```r
#1) get the estimates
estimatesuninformative <- posterior_summary(model)[c("b_age", "b_age2") , "Estimate"]
estimatesinformative   <- posterior_summary(model4)[c("b_age", "b_age2") , "Estimate"]

#2) calculate the bias
round(100*((estimatesinformative - estimatesuninformative)/estimatesuninformative), 2)
```

```
##  b_age b_age2 
## 386.09 405.69
```



_We see that the influence of this highly informative prior is around 386% and 406% on the two regression coefficients respectively. This is a large difference and we thus certainly would not end up with similar conclusions._

_The results change with different prior specifications, but are still comparable. Only using $\mathcal{N}(20, .4)$ for age, results in a really different coefficients, since this prior mean is far from the mean of the data, while its variance is quite certain. However, in general the other results are comparable. Because we use a big dataset the influence of the prior is relatively small. If one would use a smaller dataset the influence of the priors are larger. To check this you can use these lines to sample roughly 20% of all cases and redo the same analysis. The results will of course be different because we use many fewer cases (probably too few!). Use this code._


```r
indices <- sample.int(333, 60)
smalldata <- dataPHD[indices,]
```

_We made a new dataset with randomly chosen 60 of the 333 observations from the original dataset. You can repeat the analyses with the same code and only changing the name of the dataset to see the influence of priors on a smaller dataset. Run the model model.informative.priors2 with this new dataset_ 



```r
model_small_data <-  brm(formula = diff ~  age + age2, 
                                  data    = smalldata,
                                  seed    = 123)
summary(model_small_data, fit.measures = TRUE, ci = TRUE, rsquare = TRUE)
```

[/expand]

If you really want to use Bayes for your own data, we recommend to follow the [WAMBS-checklist](https://www.rensvandeschoot.com/wambs-checklist/), which you are guided through by this exercise.

--- 
### **References**

_Benjamin, D. J., Berger, J., Johannesson, M., Nosek, B. A., Wagenmakers, E.,... Johnson, V. (2017, July 22)._ [Redefine statistical significance](https://psyarxiv.com/mky9j)_. Retrieved from psyarxiv.com/mky9j_

_Greenland, S., Senn, S. J., Rothman, K. J., Carlin, J. B., Poole, C., Goodman, S. N. Altman, D. G. (2016)._ [Statistical tests, P values, confidence intervals, and power: a guide to misinterpretations](https://link.springer.com/article/10.1007/s10654-016-0149-3)_._ _European Journal of Epidemiology 31 (4_). [_https://doi.org/10.1007/s10654-016-0149-3_](https://doi.org/10.1007/s10654-016-0149-3) 

_Hoffman, M. D., & Gelman, A. (2014)_. The No-U-turn sampler: adaptively setting path lengths in Hamiltonian Monte Carlo. Journal of Machine Learning Research, 15(1), 1593-1623.

_van de Schoot R, Yerkes MA, Mouw JM, Sonneveld H (2013)_ [What Took Them So Long? Explaining PhD Delays among Doctoral Candidates](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0068839)_._ _PLoS ONE 8(7): e68839._ [https://doi.org/10.1371/journal.pone.0068839](https://doi.org/10.1371/journal.pone.0068839)

_Trafimow D, Amrhein V, Areshenkoff CN, Barrera-Causil C, Beh EJ, Bilgi? Y, Bono R, Bradley MT, Briggs WM, Cepeda-Freyre HA, Chaigneau SE, Ciocca DR, Carlos Correa J, Cousineau D, de Boer MR, Dhar SS, Dolgov I, G?mez-Benito J, Grendar M, Grice J, Guerrero-Gimenez ME, Guti?rrez A, Huedo-Medina TB, Jaffe K, Janyan A, Karimnezhad A, Korner-Nievergelt F, Kosugi K, Lachmair M, Ledesma R, Limongi R, Liuzza MT, Lombardo R, Marks M, Meinlschmidt G, Nalborczyk L, Nguyen HT, Ospina R, Perezgonzalez JD, Pfister R, Rahona JJ, Rodr?guez-Medina DA, Rom?o X, Ruiz-Fern?ndez S, Suarez I, Tegethoff M, Tejo M, ** van de Schoot R** , Vankov I, Velasco-Forero S, Wang T, Yamada Y, Zoppino FC, Marmolejo-Ramos F. (2017)_  [Manipulating the alpha level cannot cure significance testing - comments on &quot;Redefine statistical significance&quot;](https://www.rensvandeschoot.com/manipulating-alpha-level-cannot-cure-significance-testing/) PeerJ reprints 5:e3411v1   [https://doi.org/10.7287/peerj.preprints.3411v1](https://doi.org/10.7287/peerj.preprints.3411v1)

