# Bayesian meta-analysis practical

We use a Bayesian approach to meta-analysis across traits.
This considers all models of association with one trait, with two traits, and so on up to all traits.  It uses Bayes Factors to compute relative strengths of evidence.

We use data from Cotsapas et al PLoS Genetics (2011) https://doi.org/10.1371/journal.pgen.1002254
Data is effect size estimates and standard errors for 107 SNPs and 7 autoimmune traits.

## Loading the data

We load the data and get the reported effect size estimates (betas), standard errors, and P-values as follows:
```R
cotsapas = load( "practicals/cotsapas_data.RData")
rownames( cotsapas ) = cotsapas$SNP
betas = as.matrix( cotsapas[, grep( ".beta", colnames( cotsapas ))] )
ses = as.matrix( cotsapas[, grep( ".SE", colnames( cotsapas ))] )
Ps = as.matrix( cotsapas[, grep( "[.]P", colnames( cotsapas ))] )
phenotypes = gsub( ".beta", "", grep( ".beta", colnames( cotsapas ), value = T ), fixed = T )
```

## Computing Bayes factors

According to the 'Crucial Lemma' for multivariate normals shown in lectures, the product of an MVN
prior and an MVN likelihood is equal to an MVN multiplied by a constant. And the constant is itself
computable by MVN evaluated at the effect size estimate.

We use the `mvtnorm` package to compute this for any given model (specificed as a 7x7 prior covariance matrix) against the null model (under which all true effects are zero).

(This function additionally deals with missing beta estimates).

```R
compute.bf <- function( betas, ses, prior ) {
    w = which( !is.na( betas ))
    dmvnorm(
        betas[w],
        sigma = prior[w,w] + diag( ses[w]^2 )
    ) / dmvnorm(
        betas[w],
        sigma = diag( ses[w]^2 )
    )
}
```

## Choosing priors and weights

We will evaluate every possible combination of trait associations - the `expand.grid()` function
can be used to enumerate these:

```R
priors = expand.grid(
    RA = c(0,1),
    PS = c(0,1),
    MS = c(0,1),
    SLE = c(0,1),
    CD = c(0,1),
    CeD = c(0,1),
    T1D = c(0,1)
)
rownames( priors ) = sapply( 1:nrow( priors ), function(i) { paste( priors[i,], collapse = "" )})
priors = priors[-1,] # remove the null model
```

For our initial analysis we will weight each model such that there is the same prior
that k phenotypes are associated, for each k = 1,... 7.
```
prior.phenotype.counts = rowSums( priors )
weights = matrix(
    sapply( prior.phenotype.counts, function( count ) { 1 / length( which( prior.phenotype.counts == count ))}),
    nrow = 1
) / 7
```

We'll use this to compute priors for any row:
```
compute.prior.matrix <- function( row, rho = 0.9, sd = 0.2 ) {
    prior = matrix( 0, 7, 7 )
    w = which( row == 1 )
    prior[w,w] = rho * sd^2
    diag(prior)[w] = sd^2
    return( prior )
}
```

## Computing BFs
So we have to compute 127 BFs for each of 107 SNPs.  Let's do it:

```R
bfs = matrix( NA, nrow( cotsapas ), nrow( priors ),
    dimnames = list( cotsapas$SNP, rownames( priors ))
)
sd = 0.2
for( i in 1:nrow( cotsapas )) {
    beta_hat = betas[i,]
    se = ses[i,]
    for( j in 1:nrow(priors)) {
        prior = compute.prior( priors[j,] )
        bfs[i,j] = compute.bf( beta_hat, se, prior )
    }
}
```

The BFs represent evidence against the null.  Here, we aren't interested in the null model (all SNPs have been chosen because they are associated).  But we are interested in the posterior probability of each model.  We compute that by a) multiplying by the prior weights and b) normalising so that posteriors sum to 1:
```
posteriors = matrix( NA, nrow( cotsapas ), nrow( priors ),
    dimnames = list( cotsapas$SNP, rownames( priors ))
)
for( i in 1:nrow( posteriors ) ) {
    posteriors[i,] = bfs[i,] * weights
    posteriors[i,] = posteriors[i,] / sum( posteriors[i,], na.rm = T )
}
```

## Summarising the analysis

Now we can get some information out.

### Summarising by count

First, we could look at how many traits are associated in general.
To compute this we summarise by adding up the models with a given number of traits:
```R
# Compute posterior on each number of effects
byCount = matrix(
    0,
    nrow = nrow( priors ),
    ncol = 7,
    dimnames = list( rownames( priors ), 1:7 )
)
for( n in 1:7 ) {
    byCount[ which( prior.phenotype.counts == n ),i] = 1
}
count.posteriors = posteriors %*% byCount
```

For example we could find SNPs where there is evidence for more than one effect:
```R
which( sum( count.posteriors[,2:7] ) > count.posteriors[,1] )
```

### Summarising by phenotype

We could also summarise by phenotype - e.g. to find SNPs consistent with an effect for a given phenotype:

```R
byPhenotype = matrix(
    0,
    nrow = nrow( priors ),
    ncol = 7,
    dimnames = list( rownames( priors ), phenotypes )
)
for( i in 1:7 ) {
    byPhenotype[which( priors[,i] == 1 ),i] = 1
}
phenotype.posteriors = posteriors %*% byPhenotype
```

E.g. to find the expected number of associations per phenotype:
```
colSums( phenotype.posteriors )
```
(Chrohn's disease has the most)

## Interesting findings

Some SNPs are identified as having effects on all traits - yet have very non-significant P-values:

```
count.posteriors['rs11755527',]
phenotype.posteriors['rs11755527',]
betas['rs11755527',]
ses['rs11755527',]
Ps['rs11755527',]
```
What does this mean?  

Presumably, the -ve estimates for MS and CD imply that other -ve estimates appear likely under our model.  So our analysis concludes there are effects on all phenotypes.  (This is despite us assuming a +ve correlation between effects - this is not a strong condition in a MVN.)

## Caution

This analysis highlights a typical aspect of a Bayesian analysis - it requires some thought.
Here, we have a particular prior assumption that each number of associations has equal weight.
Is that realistic?

We also have a prior assumption on the correlation between traits - is that realistic?

Compare our results with those of Cotsapas et al paper.  They identified 47 SNPs with evidence of more than one association.  Their method, though, looked for statistical significance of >1 effect and didn't take into account effect size sharing.

My own view is that there are many choices we've made that need defending (or generalising).  But on the other hand the search for statistical significance is conservative

## Empirical Bayes

We could now implement an 'empirical Bayes' approach.
We learn new model weights using all the SNPs we've seen:
```R
empirical.weights = colSums( posteriors )
````
Now we could recompute BFs using the empirical weights.
