R-Parallelism in Monte Carlo Experiments using *doParallel, doRNG*
-------------------------

The *doParallel* library allows to run tasks on parallel on different
cores, rather than sequentially. The *doRNG* package extends the
previous one by enabling the replication of the whole parallel task. 

[Getting Started
Parallel](https://cran.r-project.org/web/packages/doParallel/vignettes/gettingstartedParallel.pdf)

Installation
------------

    library(doParallel)
    library(doRNG)
    library(ggplot2)
    library(dplyr)
    library(tidyr)

Example 1: Parallelizing Monte Carlo Replications
-------------------------------------------------

### Omitted variable bias in OLS estimator in linear regression model.

-   Create random data and true parameters

<!-- -->

    MonteC   <- 2000
    ## Data
    set.seed(123)
    x1       <- rnorm(1000,0,1)
    x2       <- rnorm(1000,0,1)
    x3       <- rnorm(1000,0,1)
    xx       <- matrix(data = c(x1,x2,x3),ncol = 3)      ## All variables
    xx.obs   <- xx[,1:2]                                 ## Observable variables
    # Coefficients
    btrue    <-  as.vector(x = c(0.5,1,-0.5))
    beta.hat  <- matrix(data = NA,nrow = MonteC,ncol = 2)

-   Setup Parallel Task

<!-- -->

    cores    <- detectCores()             
    cl       <- parallel::makeCluster(cores - 1, type="FORK")       # You can also select a given number of cores
    registerDoParallel(cl)                                # Register the parallel cluster
    result   <- foreach(i=1:MonteC, .combine=rbind) %dopar% {
      # Data generating process
      Y             <- xx %*% btrue + rnorm(1000,0,1)
      # Store OLS biased estimator
      beta.hat      <- solve(t(xx.obs)%*%xx.obs) %*% t(xx.obs)%*%Y
      final         <- list(b1  = beta.hat[1],
                            b2  = beta.hat[2])
    }
    parallel::stopCluster(cl)

-   Retrieve statiscs

<!-- -->

    b1 <- do.call(what = rbind, args = result[,"b1"])
    b2 <- do.call(what = rbind, args = result[,"b2"])

    empirical.b1         <- b1-btrue[1]
    true.variance        <- solve(t(xx) %*% xx)        
    true.b1              <- rnorm(n = MonteC, mean = 0, sd = sqrt(true.variance[1,1]))

-   Plot Histograms

<!-- -->

    df  <- data.frame(empirical = empirical.b1, true = true.b1)
    df2 <- df %>% gather(key,value,empirical,true)
    ggplot(df2)+
      stat_density(aes(x = value, color = key),geom = "line",position = "identity")+
      geom_vline(xintercept = 0,linetype="dashed")+
      theme_minimal()+
      theme(panel.grid.major = element_blank(),
            legend.title = element_blank())

![alt text](https://github.com/CoMoS-SA/tutorials/blob/master/Rplot01.png)


Example 2: Parallelizing Simulation Experiments
-----------------------------------------------

### Analyse the *size* of the Omitted Variable Bias 
[Little recap on OLS Regression and Omitted Variable
bias](http://www.homepages.ucl.ac.uk/~uctpsc0/Teaching/GR03/MRM.pdf)

Analitical Results lead to

*bias(β<sub>included</sub>)=β<sub>omitted</sub>\[Cov(X<sub>included</sub>, X<sub>omitted</sub>)/Var(X<sub>included</sub>)\]

If omitted variables are correlated with included variables, then we
obtained biased estimates.

 

#### Question: How correlation affect the size of the bias?

-   *We simulate 15 experiments where we let vary the degree of
    correlation of one omitted variables with one included variable*
-   We parallelize these experiments, each containing a Monte Carlo
    simulation

<!-- -->

    MonteC   <- 2000
    rho      <- seq(from = 0.5, to = 0, length.out = 15) ## Correlation coefficient
    x1       <- rnorm(1000,0,1)
    x2       <- rnorm(1000,0,1)
    x3       <- rnorm(1000,0,1)
    x        <- matrix(data = c(x1,x2,x3),ncol = 3)
    ## Uncorrelate observations
    x_c      <- x %*% solve(chol(cov(x = x)))
    zapsmall(cov(x_c))

    ##      [,1] [,2] [,3]
    ## [1,]    1    0    0
    ## [2,]    0    1    0
    ## [3,]    0    0    1

    # Coefficients
    btrue    <-  as.vector(x = c(0.5,1,-0.5))
    beta.hat  <- matrix(data = NA,nrow = MonteC,ncol = 2)

-   Setup Parallel Task

<!-- -->

    cores    <- detectCores()                             
    cl       <- makeCluster(cores - 1, type="FORK")       # You can also select a given number of cores
    registerDoParallel(cl)                                # Register the parallel cluster
    result   <- foreach(z = 1:length(rho), .combine=rbind) %dopar% {
      id.matrix      <- diag(3)
      ## We impose correlation between the omitted and one observed variable
      id.matrix[2,3] <- rho[z]
      id.matrix[3,2] <- rho[z]
      sigma          <- id.matrix
      
      Xtrue <- x_c %*% (chol(sigma))                       ## All Variables with assigned correlation
      X     <- matrix(data = Xtrue[,1:2], ncol = 2)        ## Observables with assigned correlation
      for (i in 1:MonteC){
        # Data generating process
        Y             <- Xtrue %*% btrue + rnorm(1000,0,1)
        # Store OLS biased estimator
        beta.hat[i,]  <- solve(t(X)%*%X) %*% t(X)%*%Y
      }
      ### Results for each Simulation Experiment ###
      final         <- list(b1    = beta.hat[,1],
                            b2    = beta.hat[,2],
                            Xtrue = Xtrue)
    }
    stopCluster(cl)

-   We have 15 experiments

<!-- -->

    result

    ##           b1           b2           Xtrue       
    ## result.1  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.2  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.3  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.4  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.5  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.6  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.7  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.8  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.9  Numeric,2000 Numeric,2000 Numeric,3000
    ## result.10 Numeric,2000 Numeric,2000 Numeric,3000
    ## result.11 Numeric,2000 Numeric,2000 Numeric,3000
    ## result.12 Numeric,2000 Numeric,2000 Numeric,3000
    ## result.13 Numeric,2000 Numeric,2000 Numeric,3000
    ## result.14 Numeric,2000 Numeric,2000 Numeric,3000
    ## result.15 Numeric,2000 Numeric,2000 Numeric,3000

![alt text](https://github.com/CoMoS-SA/tutorials/blob/master/Rplot.png)

#### As expected, the experiment shows the bias is linearly dependent on the correlation between the omitted variable and the included ones
![alt text](https://github.com/CoMoS-SA/tutorials/blob/master/Rplot02.png)
