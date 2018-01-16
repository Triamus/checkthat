# Description

**checkthat - A data quality framework in R**

------

# Content

* [Overview](#overview)
* [Framework design](#framework-design)
    + [Terminology](#terminology)
    + [Design ideas](#design-ideas)
* [Some initial implementation ideas](#some-initial-implementation-ideas)
    + [Working with tidyeval and expressions in the tidyverse](working-with-tidyeval-and-expressions-in-the-tidyverse)

------

# Overview

The overall idea of this package is to introduce a data quality framework which enables users to define a set of rules that any data input should adhere to. We will focus on tabular data within data frames to start with. The final package will probably be a set of functions that allow the user to detemine the number and potentially the materiality of any rule violation. The exact output variations need to be clarified.

------

# Framework design

## Terminology

### Data

When we refer to `data`, we usually mean a tabular (i.e. data frame) or columnar (i.e. vector) construct.

### Rule

When we refer to `rule` or `ruleset`, we usually mean a set of expressions to be evaluated against some data. We may use rule, expression or condition interchangeably but stick to the term expression when it concerns the programming paradigm.

### Measure

When we refer to `measure`, we usually mean a variable to quantify the extent of rule violations. This will most often be a simple count or a defined materiality measure that resides in the measured data.

### Measurement item

When we refer to `measurement item`, we usually mean the abstract notion of a rule in combination with a key/id independent of the data it may be applied to.

### Measurement

When we refer to `measurement`, we usually mean the concrete quantification/materialization of a measurement item with respect to the rule it is based on, the key/id it is mapped to and the data it is evaluated against. In addition a measurement may be connected to a timestamp to differentiate against the same measurement performed in a different point in time.

### User

When we refer to `user` or `you`, we mean a person or group of persons of any gender.

## Design ideas

### Overall concept

A user provides a set of rules in a defined `rule language` that can be parsed by functions to evaluate against a given dataset. The user receives an output that indicates the data (variable) quality in terms of some meaningful measure (e.g. count). It is to be decided whether the output is only returning an aggregated result, i.e. an aggregated measure per measured variable, or also the full evaluation result on row-basis.

### Data

Theoretically, the data could be of any shape and any type but for all practical purposes we focus on tabular data that resides in a data frame.

### Rules

We can think of rules from two perspectives, first the type of rule that is applied and second the data dimension it is applied to.

* type
    + boolean, i.e. expression evaluates to true or false
    + aggregate, i.e. expression evaluates to number (e.g. sum)
* dimension
    + row-level
    + column-level
    + dataset-level
    + group-level (grouped rows, usually only makes sense for aggregates)
    
### Extensions

One can think of a few extensions to the basic design, in particular

* an evaluation of the measurement outcome (e.g. pre-defined thresholds)
* an API framework that returns results in a web service friendly way (e.g. RESTful)

------

# Some initial implementation ideas

## Working with tidyeval and expressions in the tidyverse

To parse expressions programmatically we could make use of the [tidy evaluation](https://cran.r-project.org/web/packages/rlang/vignettes/tidy-evaluation.html) approach within the [tidyverse](https://www.tidyverse.org/) and in particular functions in [dplyr](http://dplyr.tidyverse.org/), [purrr](http://purrr.tidyverse.org/), and [rlang](https://cran.r-project.org/web/packages/rlang/index.html). If you are familiar with the [SAS Macro Language](http://support.sas.com/documentation/cdl/en/mcrolref/61885/HTML/default/viewer.htm#macro-stmt.htm) and in particular its [quoting rules](http://support.sas.com/documentation/cdl/en/mcrolref/62978/HTML/default/viewer.htm#n0o0rjikrg6iezn1ltra79iamibr.htm), you may be reminded of it in some instances although rlang is ultimately quite different. For further resources around tidy evaluation and rlang, see

* [CRAN: rlang](https://cran.r-project.org/web/packages/rlang/index.html)
* [Programming with dplyr](http://dplyr.tidyverse.org/articles/programming.html)
* [Non-standard evaluation, how tidy eval builds on base R](https://edwinth.github.io/blog/nse/)
* [Tidy evaluation, most common actions](https://edwinth.github.io/blog/dplyr-recipes/)
* [lazyeval: Non-standard evaluation](https://cran.r-project.org/web/packages/lazyeval/vignettes/lazyeval.html)
* [Advanced R: Non-standard evaluation](http://adv-r.had.co.nz/Computing-on-the-language.html)
* [Advanced R: Expressions](http://adv-r.had.co.nz/Expressions.html)
* [R Language Defiinition: Promise objects ](https://cran.r-project.org/doc/manuals/r-release/R-lang.html#Promise-objects)
* [Using nonstandard evaluation to simulate a register machine](https://tjmahr.github.io/nonstandard-eval-register-machines/)

We make up some minimal examples to showcase the overarching ideas.

```{r}
library(tidyverse)
library(rlang)

packageVersion("dplyr")
packageVersion("rlang")

data(mtcars)
```

In base R we may evaluatie some boolean expression as so.

```{r}
# base R plain boolean
mtcars[,"cyl"] > 4
mtcars$cyl > 4
```

In `rlang` we can work with `quosures` to quote our expression and leave it unevaluated until explicitly called. Note that `quosures` are environment-aware. We can evaluate via `rlang::eval_tidy()`.

```{r}
# rlang with tidy evaluation
cyl_larger_4 <- quo(mtcars$cyl > 4)
cyl_larger_4
class(cyl_larger_4)
typeof(cyl_larger_4)
rlang::eval_tidy(cyl_larger_4)

# without providing data in condition
cyl_larger_4_b <- rlang::quo("cyl" > 4)
cyl_larger_4_b
# this selects all filtered data
mtcars[rlang::eval_tidy(cyl_larger_4_b),]
```

Ultimately, we like to build functions around our calls so let's work our way towards a first working example.

```{r}
# select one variable
temp_fn <- function(x, var){
  print(rlang::enquo(var))
  var_new <- rlang::enquo(var) 
  x %>% 
    select(., rlang::UQ(var_new))
  }
temp_fn(mtcars, cyl)

# filter one variable
temp_fn <- function(x, condition){
  condition_quoted <- rlang::enquo(condition)
  x %>% 
    dplyr::filter(., rlang::eval_tidy(rlang::UQ(condition_quoted)))
  }
temp_fn(mtcars, (cyl > 4))

# transmutate one variable
temp_fn <- function(x, condition){
  condition_quoted <- rlang::enquo(condition)
  x %>% 
    dplyr::transmute(., test = rlang::eval_tidy(rlang::UQ(condition_quoted)))
  }
temp_fn(mtcars, (cyl > 4))

# transmute one variable and change output name
temp_fn <- function(x, condition, condition_name){
  condition_quoted <- rlang::enquo(condition)
  x %>% 
    dplyr::transmute(., rlang::UQ(condition_name) :=
                       rlang::eval_tidy(rlang::UQ(condition_quoted)))
  }
temp_fn(mtcars, (cyl > 4), "cyl_larger_four")
```

In the end we ideally can provide key-value pairs of named (keyed) expressions. The following design has been adapted from a discussion around [Passing named list to mutate and probably other dplyr verbs](https://community.rstudio.com/t/passing-named-list-to-mutate-and-probably-other-dplyr-verbs/2553/4).

```{r}
# multiple arguments with names
temp_fn <- function(x, args) {
  mutate(x, rlang::UQS(args))
  }

temp_fn(
  mtcars, 
  args = rlang::quos(cyl_larger_4 = cyl > 4, 
                     mpg_smaller_mean_mpg = mpg < mean(mpg)))

# also works in a dplyr chain
mtcars %>%
  temp_fn(rlang::quos(
    cyl_larger_4 = cyl > 4, 
    mpg_smaller_mean_mpg = mpg < mean(mpg)))

# we can also only keep the outcome
temp_fn <- function(x, args) {
  transmute(x, rlang::UQS(args))
  }

mtcars %>%
  temp_fn(rlang::quos(
    cyl_larger_4 = cyl > 4, 
    mpg_smaller_mean_mpg = mpg < mean(mpg)))
```

We could provide a materiality measure by introducing it as another expression via multiplying the boolean result with respective measure.

```{r}
mtcars %>%
  temp_fn(rlang::quos(
    cyl_larger_4 = cyl > 4,
    cyl_larger_4_mat = (cyl > 4) * mpg,
    mpg_smaller_mean_mpg = mpg < mean(mpg)))
```

Finally, we may only be interested in the overall result rather than row-wise outcomes.

```{r}
mtcars %>%
  temp_fn(rlang::quos(
    cyl_larger_4 = cyl > 4,
    cyl_larger_4_mat = (cyl > 4) * mpg,
    mpg_smaller_mean_mpg = mpg < mean(mpg))) %>%
  summarise(cyl_larger_4 = sum(cyl_larger_4, na.rm = TRUE),
            cyl_larger_4_mat = sum(cyl_larger_4_mat, na.rm = TRUE),
            mpg_smaller_mean_mpg = sum(mpg_smaller_mean_mpg, na.rm = TRUE))
```

