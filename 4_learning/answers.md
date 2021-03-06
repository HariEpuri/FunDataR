# Fundamentals of computational data analysis using R
## Machine learning
#### Contact: mitch.kostich@jax.org

---

### Index

- [Lesson 1 : Check 1](#lesson-1--check-1)
- [Lesson 1 : Check 2](#lesson-1--check-2)
- [Lesson 2 : Check 1](#lesson-2--check-1)
- [Lesson 2 : Check 2](#lesson-2--check-2)
- [Lesson 2 : Check 3](#lesson-2--check-3)
- [Lesson 3 : Check 1](#lesson-3--check-1)
- [Lesson 3 : Check 2](#lesson-3--check-2)
- [Lesson 3 : Check 3](#lesson-3--check-3)

---

### Lesson 1 : Check 1 

Use the code above as a guide to use the 1-se rule and performance estimates (Cohen's Kappa) based
  on 7-fold cross-validation repeated 3 times to choose the value of `k` (number of nearest neighbors) 
  to use when predicting `Species` based only on the two features `Sepal.Width` and `Sepal.Length`. Try
  the following values for `k`: `c(1, 3, 5, 11, 15, 21, 31, 51, 75)`.

```
library('caret')
library('class')

rm(list=ls())

## prep data:
dat <- iris[, c('Species', 'Sepal.Width', 'Sepal.Length')]
summary(dat)

## inner cross-validation function; used to evaluate kappa for k neighbors using 
##   one fold defined by idx.trn:

f.cv <- function(idx.trn, dat, k) {
  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]
  ## NOTE: change in index of Species column from 5 to 1:
  prd.tst <- class::knn(train=dat.trn[, -1], test=dat.tst[, -1], cl=dat.trn[, 1], k=k)
  kap.tst <- caret::confusionMatrix(dat.tst[, 1], prd.tst)$overall[['Kappa']]
  kap.tst
}

## calls the f.cv() inner cross-validation function for different numbers of neighbors 
##   specified in ks and fold specified by idx.trn:

f.cv.cmp <- function(idx.trn, dat, ks) {
  rslt <- rep(as.numeric(NA), length(ks))
  for(i in 1:length(ks)) {
    rslt[i] <- f.cv(idx.trn, dat=dat, k=ks[i])
  }
  names(rslt) <- ks
  rslt
}

## generate folds:
set.seed(1)
idx <- 1:nrow(dat)
folds <- caret::createMultiFolds(idx, k=7, times=3)

## evaluate different values for k:

ks <- c(1, 3, 5, 11, 15, 21, 31, 51, 75)
rslt <- sapply(folds, f.cv.cmp, dat=dat, ks=ks)

## process the results:

m <- apply(rslt, 1, mean)
s <- apply(rslt, 1, sd)
se <- s / sqrt(nrow(rslt))
(rslt <- data.frame(k=ks, kap.mean=m, kap.se=se))

## the 'best' result:

(idx.max <- which.max(m))
rslt[idx.max, ]

## apply 1-se rule to attenuate potential overfitting:

(cutoff <- m[idx.max] - se[idx.max])
i.good <- m >= cutoff
rslt[i.good, ]
max(ks[i.good])
i.pick <- ks == max(ks[i.good])
rslt[i.pick, ]

```

[Return to index](#index)

---

### Lesson 1 : Check 2

Use the code from the above example as a template to conduct a nested cross-validation 
  of the knn-classification of the `iris` data `Species` based only on the two features 
  `Sepal.Width` and `Sepal.Length`. Use 7-fold CV, repeated 3-times at both levels, using 
  the inner-CV to select a value for `k` and the outer-CV to evaluate the entire procedure.
  Express performance in terms of Cohen's Kappa. Calculate the standard error of the 
  final performance estimate.

```
library('caret')
library('class')

rm(list=ls())

## prep data:
dat <- iris[, c('Species', 'Sepal.Width', 'Sepal.Length')]
summary(dat)

## inner cross-validation function; used to evaluate kappa for k neighbors using 
##   one fold defined by idx.trn:

f.cv <- function(idx.trn, dat, k) {
  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]
  prd.tst <- class::knn(train=dat.trn[, -1], test=dat.tst[, -1], cl=dat.trn[, 1], k=k)
  kap.tst <- caret::confusionMatrix(dat.tst[, 1], prd.tst)$overall[['Kappa']]
  kap.tst
}

## calls the f.cv() inner cross-validation function for different numbers of neighbors 
##   specified in ks and fold specified by idx.trn:

f.cv.cmp <- function(idx.trn, dat, ks) {
  rslt <- rep(as.numeric(NA), length(ks))
  for(i in 1:length(ks)) {
    rslt[i] <- f.cv(idx.trn, dat=dat, k=ks[i])
  }
  names(rslt) <- ks
  rslt
}

## uses the inner cross-validation to pick the value for the number of neighbors 
##   from the selection specified in ks:

f.pick.k <- function(dat, ks) {

  ## folds for inner cross-validation (used for tuning parameter)
  idx <- 1:nrow(dat)
  folds <- caret::createMultiFolds(idx, k=7, times=3)
  rslt <- sapply(folds, f.cv.cmp, dat=dat, ks=ks)

  m <- apply(rslt, 1, mean)
  s <- apply(rslt, 1, sd)
  se <- s / sqrt(nrow(rslt))

  ## the 'best' result:
  idx.max <- which.max(m)

  ## apply 1se rule to attenuate potential overfitting:
  cutoff <- m[idx.max] - se[idx.max]
  i.good <- m >= cutoff
  max(ks[i.good])
}

f.cv.outer <- function(idx.trn, dat) {
  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]
  ks <- c(1, 3, 5, 10, 15, 20, 30, 50, 75)
  k.pick <- f.pick.k(dat=dat.trn, ks=ks)
  prd.tst <- class::knn(train=dat.trn[, -1], test=dat.tst[, -1], cl=dat.trn[, 1], k=k.pick)
  caret::confusionMatrix(dat.tst[, 1], prd.tst)$overall[['Kappa']]
}

set.seed(1)
idx <- 1:nrow(iris)
folds <- caret::createMultiFolds(idx, k=7, times=3)
rslt <- sapply(folds, f.cv.outer, dat=dat)
mean(rslt)                       ## the final performance estimate
sd(rslt)                         ## standard deviation of performance results
sd(rslt) / sqrt(length(rslt))    ## standard error of performance estimate

```

[Return to index](#index)

---

### Lesson 2 : Check 1

Start with the following dataset, which has 100 extraneous variables added:

```
rm(list=ls())

set.seed(1)
dat <- mtcars
for(i in 1:100) {
  nom <- paste('s', i, sep='')
  dat[[nom]] <- rnorm(nrow(dat), 0, 10)
}

```

Use the example from the end of the last section to conduct a 5-fold cross-validation, 
  repeated 3-times to compare the MSEs for:

1) a model derived using `lm()` and `step()`, with the scope between `mpg ~ 1` and `mpg ~ .`;

2) a ridge regression model (with lambda tuning per the example) with `mpg` as response and
   all other variables as predictors;

3) a lasso regression model (with lambda tuning per the example) with `mpg` as response and
   all other variables as predictors;

Make sure to use the same folds for all three procedures.

```
library(glmnet)
library(caret)

rm(list=ls())

set.seed(1)
dat <- mtcars
for(i in 1:100) {
  nom <- paste('s', i, sep='')
  dat[[nom]] <- rnorm(nrow(dat), 0, 10)
}

set.seed(1)
idx <- 1:nrow(dat)
folds <- caret::createMultiFolds(idx, k=5, times=3)

f.cv <- function(idx.trn) {

  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]

  fit.lm.lo <- lm(mpg ~ 1, data=dat.trn)
  fit.lm.hi <- lm(mpg ~ ., data=dat.trn)
  fit.lm <- step(fit.lm.lo, scope=list(lower=fit.lm.lo, upper=fit.lm.hi), direction='both', trace=1)

  cv.ridge <- cv.glmnet(x=as.matrix(dat.trn[, -1]), y=dat.trn[, 1], alpha=0)
  cv.lasso <- cv.glmnet(x=as.matrix(dat.trn[, -1]), y=dat.trn[, 1], alpha=1)

  fit.ridge <- cv.ridge$glmnet.fit
  fit.lasso <- cv.lasso$glmnet.fit

  y.lm <- predict(fit.lm, newdata=dat.tst[, -1]) 
  y.ridge <- predict(fit.ridge, newx=as.matrix(dat.tst[, -1]), s=cv.ridge$lambda.min)
  y.lasso <- predict(fit.lasso, newx=as.matrix(dat.tst[, -1]), s=cv.lasso$lambda.min)

  mse.lm <- mean((y.lm - dat.tst[, 1]) ^ 2)
  mse.ridge <- mean((y.ridge - dat.tst[, 1]) ^ 2)
  mse.lasso <- mean((y.lasso - dat.tst[, 1]) ^ 2)

  c(lm=mse.lm, ridge=mse.ridge, lasso=mse.lasso)
}

idx.trn <- folds[[1]]
f.cv(idx.trn)

rslts <- sapply(folds, f.cv)
apply(rslts, 1, mean)
apply(rslts, 1, sd) / sqrt(length(folds))

```

[Return to index](#index)

---

### Lesson 2 : Check 2

Starting with the following dataset (where noise has been added to the predictors):

```
library(glmnet)
library(caret)

rm(list=ls())

data(tecator)
dat <- absorp

dat <- t(apply(dat, 1, function(v) (v - mean(v)) / sd(v)))
dat <- t(apply(dat, 1, function(v) v + rnorm(length(v), 0, 0.5)))
dat <- cbind(endpoints[, 1], dat)
dat <- data.frame(dat)
names(dat) <- c('y', paste('x', 1:ncol(absorp), sep=''))

```

Use the example from the end of the last section to set up a 5-fold cross-validation, 
  repeated 3-times, to compare the MSEs from:

1) a model derived using `lm()` and `step()`, with the scope between `y ~ 1` and `mpg ~ .`;

2) elastic-net models with alpha set to the following values `0, 0.2, 0.4, 0.6, 0.8, 1`, 
   tuning `lambda` in each case.

Make sure to use the same folds for each model.

```
library(glmnet)
library(caret)

rm(list=ls())

data(tecator)
dat <- absorp

dat <- t(apply(dat, 1, function(v) (v - mean(v)) / sd(v)))
dat <- t(apply(dat, 1, function(v) v + rnorm(length(v), 0, 0.5)))
dat <- cbind(endpoints[, 1], dat)
dat <- data.frame(dat)
names(dat) <- c('y', paste('x', 1:ncol(absorp), sep=''))

## generate folds:

set.seed(1)
idx <- 1 : nrow(dat)
folds <- caret::createMultiFolds(idx, k=5, times=3)

## function for cross-validation: note use of internal functions:

f.cv <- function(idx.trn) {

  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]

  fit.lm.lo <- lm(y ~ 1, data=data.frame(dat.trn))
  fit.lm.hi <- lm(y ~ ., data=data.frame(dat.trn))
  fit.lm <- step(fit.lm.lo, scope=list(lower=fit.lm.lo, upper=fit.lm.hi), direction='both', trace=1)

  f.alpha <- function(alpha.i) {
    cv.glmnet(x=as.matrix(dat.trn[, -1]), y=dat.trn[, 1], alpha=alpha.i)
  }

  alphas <- seq(from=0, to=1, by=0.2)
  cvs <- lapply(alphas, f.alpha)
  names(cvs) <- paste('s', alphas, sep='')

  f.fit <- function(cv.i) cv.i$glmnet.fit
  fits <- lapply(cvs, f.fit)

  f.prd <- function(idx) {
    predict(fits[[idx]], newx=as.matrix(dat.tst[, -1]), s=cvs[[idx]]$lambda.min)
  }
  ys <- sapply(1:length(fits), f.prd)
  y.lm <- predict(fit.lm, newdata=dat.tst[, -1]) 

  f.mse <- function(y.i) mean((y.i - dat.tst[, 1]) ^ 2)
  mses <- apply(ys, 2, f.mse)
  names(mses) <- names(cvs)
  c(mses, lm=f.mse(y.lm))
}

rslts <- sapply(folds, f.cv)
apply(rslts, 1, mean)
apply(rslts, 1, sd) / sqrt(length(folds))

```

[Return to index](#index)

---

### Lesson 2 : Check 3

Use the example from the end of the last section to set up a 5-fold cross-validation, 
  repeated 1-time (because SVM tuning using one thread on a laptop is slow), to compare 
  the AUCs (make sure to get back class probabilities instead of assignments from `predict()`:

1) an elastic-net logistic regression model with `alpha=0.5` and `lambda` tuning, with 
   `dhfr[, 1]` as the response and all other columns as the predictors. 

2) an SVM logistic regression model with tuning of gamma within the range `2^(-5:3)`, 
   and tuning of cost within the range `2^(-3:5)`.

Make sure to use the same folds for each model.

```
library(caret)
library(pROC)
library(glmnet)
library(e1071)

data(dhfr)                        ## from caret
dat <- dhfr

f.cv <- function(idx.trn) {

  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]

  cv.net <- glmnet::cv.glmnet(x=as.matrix(dat.trn[, -1]), y=dat.trn[, 1], alpha=0.5, family='binomial')
  fit.net <- cv.net$glmnet.fit
  prd.net <- predict(fit.net, newx=as.matrix(dat.tst[, -1]), s=cv.net$lambda.min, type='response')
  prd.net <- prd.net[, 1]

  cv.svm <- e1071::tune.svm(Y ~ ., data=dat.trn, gamma=2^(-5:3), cost=2^(-3:5), probability=F)
  fit.svm <- e1071::svm(Y ~ ., data=dat.trn, probability=T, gamma=cv.svm$best.parameters['gamma'], cost=cv.svm$best.parameters['cost'])

  prd.svm <- predict(fit.svm, newdata=dat.tst[, -1], probability=T)
  prd.svm <- attr(prd.svm, 'probabilities')[, 'active']

  auc.net <- as.numeric(pROC::roc(dat.tst$Y == 'active', prd.net, direction='>')$auc)
  auc.svm <- as.numeric(pROC::roc(dat.tst$Y == 'active', prd.svm, direction='<')$auc)

  c(glmnet=auc.net, svm=auc.svm)
}

set.seed(1)
idx <- 1 : nrow(dat)
folds <- caret::createMultiFolds(idx, k=5, times=1)

f.cv(folds[[1]])

(rslts <- sapply(folds, f.cv))
apply(rslts, 1, mean)
apply(rslts, 1, sd) / sqrt(length(folds))

```

[Return to index](#index)

---

### Lesson 3 : Check 1

Starting with the following dataset:

```
library(rpart)
library(caret)

rm(list=ls())

## reformat prostate cancer recurrence dataset:
dat <- rpart::stagec
dat$pgstat <- c('no', 'yes')[dat$pgstat + 1]
dat$pgstat <- factor(dat$pgstat)  ## character -> factor
dat$pgtime <- NULL                ## drop column

```

Using 5-fold cross-validation repeated 12 times, generate a point estimate of the AUC
  for an `rpart` classification tree with formula `pgstat ~ .` where the `cp` complexity
  parameter is chosen using an inner cross-validation loop. Hint: you don't need to
  do the inner cross-validation explicitly -- `rpart()` does it for you.

```
library(rpart)
library(caret)

rm(list=ls())

## reformat prostate cancer recurrence dataset:

dat <- rpart::stagec
dat$pgstat <- c('no', 'yes')[dat$pgstat + 1]
dat$pgstat <- factor(dat$pgstat)  ## character -> factor
dat$pgtime <- NULL                ## drop column

f.cv <- function(idx.trn) {
  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]

  ## fit the model, tuning 'cp' complexity parameter by (10-fold) CV:
  fit0 <- rpart(pgstat ~ ., data=dat.trn, method='class')
  tbl <- fit0$cptable

  ## prune tree using 'cp' value chosen from tuning:
  idx.bst <- which.min(tbl[, 'CP'])
  fit <- prune(fit0, cp=tbl[idx.bst, 'CP'])

  ## make (probabilistic) predictions and take a look:
  prd.tst <- predict(fit, newdata=dat.tst, type='prob') 
  prd.tst <- prd.tst[, 'yes']
  roc.tst <- pROC::roc(dat.tst$pgstat == 'yes', prd.tst, direction='<')
  roc.tst$auc
}

set.seed(1)
idx <- 1:nrow(dat)
folds <- caret::createMultiFolds(idx, k=5, times=12)

## test function:
f.cv(folds[[1]])

rslts <- sapply(folds, f.cv)
mean(rslts)
sum(rslts <= 0.5)
sum(rslts <= 0.5) / length(rslts)

```

[Return to index](#index)

---

### Lesson 3 : Check 2

Use the `dhfr` data from the `caret` package to perform 5-fold cross-validation repeated twice to 
  estimate the AUC for a model with `Y` as categorical response and the rest of the features as
  predictors. Tune the `mtry` parameter using the `tuneRF()` function, specifying `stepFactor=0.5` 
  and `improve=0.01`.

```
library(caret)
library(pROC)
library(randomForest)

## data:

rm(list=ls())
data(dhfr)                        ## from caret
dat <- dhfr

f.cv <- function(idx.trn) {

  ## split into training and test-sets:
  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]

  ## tuning mtry:
  tune.fit <- tuneRF(x=dat.trn[, -1], y=dat.trn$Y, stepFactor=0.5, improve=0.01, 
    ntreeTry=1000, trace=F, plot=F, doBest=F, replace=T)

  score.best <- min(tune.fit[, 'OOBError'])
  i.best <- tune.fit[, 'OOBError'] == score.best
  mtry.best <- min(tune.fit[i.best, 'mtry'])

  ## fit with selected mtry:
  fit <- randomForest(x=dat.trn[, -1], y=dat.trn$Y, mtry=mtry.best, ntree=1000, importance=T, replace=T)

  ## probabilistic predictions:
  prd.tst <- predict(fit, newdata=dat.tst[, -1], type='prob')
  prd.tst <- prd.tst[, 'active']

  ## evaluation: 
  roc.tst <- pROC::roc(dat.tst$Y == 'active', prd.tst, direction='<')
  roc.tst$auc
}

set.seed(1)
idx <- 1 : nrow(dat)
folds <- caret::createMultiFolds(idx, k=5, times=2)

rslts <- sapply(folds, f.cv)
mean(rslts)
range(rslts)

```

[Return to index](#index)

---

### Lesson 3 : Check 3

Starting with the following data:

```
library(rpart)

rm(list=ls())

## reformat prostate cancer recurrence dataset:
dat <- rpart::stagec
dat$pgtime <- NULL                ## drop column

```

Use 5-fold cross-validation, repeated three times to estimate the AUC of a gradient boosted 
  tree model, built using `gbm()` with `shrinkage=0.01`, `interaction.depth=2`, and 
  `n.minobsinnode=5`. Use an inner 5-fold cross-validation loop to select the number of
  iterations. Hint: the `gbm()` function does the inner-loop parameter tuning for you.

```
library(caret)
library(gbm)

f.cv <- function(idx.trn) {

  ## split data into training and test-sets:

  dat.trn <- dat[idx.trn, ]
  dat.tst <- dat[-idx.trn, ]

  ## fit gbm model to training-set:

  fit <- gbm::gbm(pgstat ~ ., data=dat.trn, distribution="bernoulli", 
    n.trees=1000, shrinkage=0.01, interaction.depth=2, n.minobsinnode=5, 
    cv.folds=5, keep.data=F, verbose=T, n.cores=1)

  ## best stopping point; no plots needed:

  n.trees.best <- gbm.perf(fit, method="cv", plot.it=F)

  ## evaluate probabilistic predictions:

  prd <- predict(fit, newdata=dat.tst, n.trees=n.trees.best, type="response")
  roc.tst <- pROC::roc(dat.tst$pgstat, prd, direction='<')
  roc.tst$auc
}

set.seed(1)
idx <- 1 : nrow(dat)
folds <- caret::createMultiFolds(idx, k=5, times=3)

rslts <- sapply(folds, f.cv)
mean(rslts)
range(rslts)

```

[Return to index](#index)

---

## FIN!
