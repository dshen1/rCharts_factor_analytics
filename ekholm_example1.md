---
title: Explore Anders Ekholm's SelectionShare & TimingShare
author: Timely Portfolio
github: {user: timelyportfolio, repo: rCharts_factor_analytics, branch: "gh-pages"}
framework: bootplus
layout: post
mode: selfcontained
highlighter: prettify
hitheme: twitter-bootstrap
lead : >
  Simple Example in R
assets:
  js:
    - "http://d3js.org/d3.v3.min.js"
    - "http://dimplejs.org/dist/dimple.v2.0.0.min.js"
    - "http://timelyportfolio.github.io/rCharts_dimple/js/d3-grid.js"
  css:
    - "http://fonts.googleapis.com/css?family=Raleway:300"
    - "http://fonts.googleapis.com/css?family=Oxygen"    
---
# SelectionShare & TimingShare


<style>
body{
  font-family: 'Oxygen', sans-serif;
  font-size: 15px;
  line-height: 22px;
}

h1,h2,h3,h4 {
  font-family: 'Raleway', sans-serif;
}

</style>


[Petajisto and Cremers' (2009)](ssrn.com/abstract=891719) ActiveShare and Tracking Error decomposition of money manager returns made what I consider to be revolutionary discoveries, but unfortunately are incredibly costly/difficult to calculate on mutual funds since they require holdings-level data.  In his latest two papers, [Anders Ekholm](www.andersekholm.fi) demonstrates how to similarly decompose performance armed only with the return stream of the manager.  His SelectionShare and TimingShare metrics are both an ingenious standalone contribution and a valuable indirect replication/validation of Petajisto/Cremers.  In case my ability to reword/summarize is not sufficient, I'll include the following quote from Ekholm summarizing the research.

<blockquote>
"Cremers andPetajisto (2009) and Petajisto (2013) find that past ActiveShare is positively related to future
performance. Ekholm (2012) takes a different approach and shows that the excess risk caused by selectivity and timing can be estimated from portfolio returns...
<br/><br/>
We develop the methodology presented by Ekholm (2012) one step further, and present two
new measures that quantify how much selectivity and timing have contributed to total variance.
Our SelectionShare and TimingShare measures can be estimated using portfolio returns only, which has both theoretical and practical advantages. Our empirical tests show that all active risk is not equal, as selectivity and timing have opposite effects on performance."
</blockquote>

Below with a little replicable R code, I will extend Ekholm's [example spreadsheet](http://www.andersekholm.fi/selection_timing/) to a real fund and calculate ActiveAlpha and ActiveBeta (2009 published 2012).  Then as Ekholm does, use these ActiveAlpha and ActiveBeta metrics to get SelectionShare and TimingShare (2014).  Since these calculations are just basic linear regression, I think it is well within the scope of nearly all readers' abilities.



---
# References in Code Comments

I am not sure that this will be helpful, and I have not done this in the past, but I will include references to the research within code comments.  For someone reading the code and not the content, this will insure that these links do not get lost.


```r
# perform Ekholm (2012,2014) analysis on mutual fund return data

# Ekholm, A.G., 2012
# Portfolio returns and manager activity:
#    How to decompose tracking error into security selection and market timing
# Journal of Empirical Finance, Volume 19, pp 349-358

# Ekholm, Anders G., July 21, 2014
# Components of Portfolio Variance:
#    R2, SelectionShare and TimingShare
# Available at SSRN: http://ssrn.com/abstract=2463649
```

---
# Depend on Other R Packages

We will, as always, depend on the wonderful and generous contributions of others in the form of R packages.  Most of the calculations though are just the base `lm(...)`.  I do not think there is any turning back from the pipes in [`magrittr`](cran.r-project.org/package=magrittr) or [`pipeR`](renkun.me/pipeR-tutorial/), and since I am addicted I will afford myself the luxury of pipes.


```r
require(quantmod)
require(PerformanceAnalytics)
# love me some pipes; will happily translate if pipes aren't your thing
require(pipeR) 
```

---
# Ugly Way to Get Kenneth French Factor Data

Get the full set of Fama/French factors from the very generous [Kenneth French data library](mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html).  This example only perform a Jensen regression, so we will only need the `Mkt.RF`.  However, in future installments, we will do the Carhart regression which requires the full factor set.


```r
#daily factors from Kenneth French Data Library
#get Mkt.RF, SMB, HML, and RF
#UMD is in a different file
my.url="http://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_Factors_daily.zip"
my.tempfile<-paste(tempdir(),"\\frenchfactors.zip",sep="")
paste(tempdir(),"\\F-F_Research_Data_Factors_daily.txt",sep="") %>>%
  (~ download.file( my.url, my.tempfile, method="auto", 
              quiet = FALSE, mode = "wb",cacheOK = TRUE )
  ) %>>%
  (~ unzip(my.tempfile,exdir=tempdir(),junkpath=TRUE ) ) %>>%
  (
    #read space delimited text file extracted from zip
    read.table(file= . ,header = TRUE, sep = "", as.is = TRUE,
                 skip = 4, nrows=23257)
  ) %>>%
  (
    as.xts( ., order.by=as.Date(rownames(.),format="%Y%m%d" ) )
  ) -> french_factors_xts

#now get the momentum factor
my.url="http://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Momentum_Factor_daily.zip"
my.usefile<-paste(tempdir(),"\\F-F_Momentum_Factor_daily.txt",sep="")
download.file(my.url, my.tempfile, method="auto", 
              quiet = FALSE, mode = "wb",cacheOK = TRUE)
unzip(my.tempfile,exdir=tempdir(),junkpath=TRUE)
#read space delimited text file extracted from zip
read.table(file=my.usefile, header = TRUE, sep = "",
              as.is = TRUE, skip = 13, nrows=23156) %>>%
  ( #get xts for analysis    
    as.xts( . , order.by=as.Date( rownames(.), format="%Y%m%d"  ) )
  ) %>>%
  #merge UMD (momentum) with other french factors
  ( merge( french_factors_xts, . ) )  %>>%
  na.omit %>>%
  ( .[] / 100 ) %>>%
  (~ plot.zoo(.) ) -> french_factors_xts
```

---
# Get Fund Data from Yahoo! Finance


```r
#get a fund to analyze
# will use Vulcan Value the biggest mutual fund in Birmingham, Alabama
ticker <- 'VVPLX'
ticker %>>% 
  getSymbols( from="1896-01-01", adjust=TRUE, auto.assign=F ) %>>%
  ( .[,4] ) %>>%
  ROC( type = "discrete", n = 1 ) %>>%
  merge ( french_factors_xts ) %>>%
  na.omit -> perfComp

colnames(perfComp)[1] <- gsub( ".Close", "", colnames(perfComp)[1] )
perfComp %>>% plot.zoo ( main = paste0(ticker, " with Factors" ) )
```

![plot of chunk mutualfund_data](assets/fig/mutualfund_data.png) 


---
# Calculate Ekholm's SelectionShare and TimingShare


```r
# do it with lots of comments and no pipes
# to clarify the steps

# 1.  Linear Regression of Fund Return vs (Market - RiskFree)
#      which gives us the well-known Jensen alpha and beta
jensenLM <- lm( data = perfComp, VVPLX ~ Mkt.RF )

# 2.  Run another linear regression on the residuals ^2
#       vs the (Mkt - Rf)^2
residuals.df <- data.frame(
  residuals = as.numeric( residuals( jensenLM ) ) ^ 2
  , Mkt.RF_sq = as.numeric( perfComp$Mkt.RF ^ 2 )
)
residualsLM <- lm(
  data = residuals.df
  , residuals.df$residuals ~ residuals.df$Mkt.RF_sq
)

# 3. Get ActiveAlpha and ActiveBeta from coefficients
#     see
# Ekholm, A.G., 2012
# Portfolio returns and manager activity:
#    How to decompose tracking error into security selection and market timing
# Journal of Empirical Finance, Volume 19, pp 349-358
activeAlpha = coefficients( residualsLM )[1] ^ (1/2)
activeBeta = coefficients( residualsLM )[2] ^ (1/2)

# 4. Last step to calculate SelectionShare and TimingShare
selectionShare = activeAlpha ^ 2 /
                    (
                      var( perfComp$VVPLX ) *
                      (nrow( perfComp ) - 1) / nrow( perfComp )
                    )

timingShare = activeBeta ^ 2 *
                mean( residuals.df$Mkt.RF_sq ) /
                (
                  var( perfComp$VVPLX ) *
                    ( nrow( perfComp ) - 1) / nrow( perfComp )
                )

# check our work r^2  + selectionShare + timingShare should equal 1
summary(jensenLM)$"r.squared" + selectionShare + timingShare
```

      VVPLX
VVPLX     1

---
# One-liner with Function


```r
jensen_ekholm <- function( data, ticker = NULL ){
  
  if(is.null(ticker)) ticker <- colnames(data)[1]
  
  as.formula ( paste0(ticker, " ~  Mkt.RF" ) ) %>>%
    ( lm( data = data, . ) -> jensenLM )
  
  jensenLM %>>%
    residuals %>>%
    (. ^ 2 ) %>>%
    (
      data.frame(
        data
        , "fitted_sq" = .
        , lapply(data[,2],function(x){
          structure(
            data.frame( as.numeric(x) ^ 2 )
            , names = paste0(names(x),"_sq")
          ) %>>%
            return
        }) %>>% ( do.call( cbind, . ) )
      ) -> return_data_jensen
    )
  
  return_data_jensen %>>%
    ( lm( fitted_sq ~ Mkt.RF_sq, data = . ) )%>>%
    coefficients %>>%
    ( . ^ (1/2) ) %>>%
    t %>>%
    (
      structure(
        data.frame(.),
        names = c("ActiveAlpha", paste0("ActiveBeta_",colnames(.)[-1]))
      )
    ) %>>% 
    (
      data.frame(
        .
        , "SelectionShare" = .$ActiveAlpha ^ 2 / (var(return_data_jensen[,ticker]) * (nrow(return_data_jensen) - 1) / nrow(return_data_jensen))
        , "TimingShare" = .$ActiveBeta_Mkt.RF_sq ^ 2* mean( return_data_jensen$Mkt.RF_sq ) / (var(return_data_jensen[,ticker]) * (nrow(return_data_jensen) - 1) / nrow(return_data_jensen))
        
      )
    ) %>>%
    (
      list( "ekholm" = ., "linmod" = jensenLM )
    ) %>>%
    return
}

jensen_ekholm( perfComp ) -> jE

jE %>>% ( summary(.$linmod)$"r.squared" + jE$ekholm[1,3] + jE$ekholm[1,4] )
```

[1] 1