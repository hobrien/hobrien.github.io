## Logical operators

- Use & and | to create logical vectors for subsetting (element-wise comparison):

```{r}
x<-1:5
x[x<5 & x>1] # good
#> [1] 2 3 4

x[x<5 && x>1] # bad
#> integer(0)
```

- Use && and || in if statements (rhs not evaluated if lhs enough to determine result)

```{r}
x<-'foo'
if ( is.numeric(x) && x %% 2 == 0 ) "even number" # good

if ( is.numeric(x) & x %% 2 == 0 ) "even number " # bad
#> Error in x%%2 : non-numeric argument to binary operator
```

## Conditional mutate

- Use ifelse() for binary conditions
```{r}
df<-data.frame(numbers=sample(1:10,5))
df %>% mutate(type = ifelse(x %% 2 == 0, 'even', 'odd'))

#>   numbers type
#> 1       3 even
#> 2      10 even
#> 3       6 even
#> 4       8 even
#> 5       1 even
```

- Use dplyr::case_when() for small number of options

```{r}
df<-data.frame(numbers=sample(1:10,5))
df %>% mutate(type = case_when(numbers < 3 ~ "low",
                               numbers < 7 ~ "intermediate",
                               TRUE ~ "high"
                               ) 
              )
#>   numbers         type
#> 1       1          low
#> 2       5 intermediate
#> 3       9         high
#> 4       2          low
#> 5       3 intermediate

# conditions are evaluated in order until one is satisfied
```

- Use dplyr::left_join(lookup_table) for large number of options

```{r}
lookup_table<-data.frame(numbers=1:10, 
                   words=c("one", "two", "three", "four", "five", "six", "seven", "eight", "nine", "ten")
                   )
df %>% left_join(lookup_table)
#> Joining, by = "numbers"
#>   numbers words
#> 1       6   six
#> 2      10   ten
#> 3       2   two
#> 4       4  four
#> 5       1   one
```

## Add elements from complex data structures to dataframe

- dplyr::mutate works by feeding specified columns into vectorized function and adding the output as a new column.
- this means that it won't work with non-vectorized functions:

```{r}
df <- data.frame(success=c(5,8,4), total=c(20,21,22))

mutate(df, p_value=binom.test(success, total)$"p.value") # bad
#> Error in mutate_impl(.data, dots) : 
#>   Evaluation error: incorrect length of 'x'.
```

- mutate can be forced to run the function independently on each row by applying ```dplyr::rowwise()``` to the dataframe:

```{r}
df <- data.frame(success=c(5,8,4), total=c(20,21,22))

mutate(rowwise(df), p_value=binom.test(success, total)$"p.value") # better
#> Source: local data frame [3 x 3]
#> Groups: <by row>

#> # A tibble: 3 x 3
#>   success total    p_value
#>     <dbl> <dbl>      <dbl>
#> 1       5    20 0.04138947
#> 2       8    21 0.38331032
#> 3       4    22 0.00434351
```

- a more efficient option is to create a vectorized function by using ```purrr::map```/```purrr::map2```:

```{r}
df <- data.frame(success=c(5,8,4), total=c(20,21,22))

mutate(df, binom = map2(success, total, binom.test), p_value=map_dbl(binom, 'p.value')) %>% select(-binom) # best
#>   success total    p_value
#> 1       5    20 0.04138947
#> 2       8    21 0.38331032
#> 3       4    22 0.00434351
```
- this also makes it possible to pull out multiple elements without rerunning the function that created the complex data structure:
```{r}
df <- data.frame(success=c(5,8,4), total=c(20,21,22))

mutate(df, binom = map2(success, total, binom.test), 
           p_value=map_dbl(binom, 'p.value'), 
           proportion=map_dbl(binom, 'estimate')) %>% 
  select(-binom)
#>   success total    p_value proportion
#> 1       5    20 0.04138947  0.2500000
#> 2       8    21 0.38331032  0.3809524
#> 3       4    22 0.00434351  0.1818182
```

## plot on log-log scale
- plot x and y variables on log-log scale with log-log grid lines
- there's probably a way to use tidyeval so I wouldn't have to quote the variable names, but I'm not maeesing with this now
- there's also code to include a best-fit line, but it only seems to work when x has multiple observations with exactly the same value so I've commented it out
- uses a handy function called `aes_string` to deal with passing column names as arguments

```{r}
library(tidyverse)
library(scales)
log_plot <- function(df, x_var, y_var, group_var) {
  log10_breaks <- trans_breaks("log10", function(x) 10 ^ x)
  log10_mbreaks <- function(x) {
    limits <- c(floor(log10(x[1])), ceiling(log10(x[2])))
    breaks <- 10 ^ seq(limits[1], limits[2])

    unlist(lapply(breaks, function(x) x * seq(0.1, 0.9, by = 0.1)))
  }
  log10_labels <- trans_format("log10", math_format(10 ^ .x))

  ggplot(df, aes_string(x = x_var, y =y_var, colour = group_var)) +
    geom_point() +
    #stat_summary(aes_string(group = group_var), fun.y = mean, geom = "line") +
    scale_y_log10(
      breaks = log10_breaks, labels = log10_labels, minor_breaks = log10_mbreaks
    ) +
    scale_x_log10(
      breaks = log10_breaks, labels = log10_labels, minor_breaks = log10_mbreaks
    ) +
    theme_bw() +
    theme(aspect.ratio = 1, legend.justification = "top")
}

diamonds %>% filter(color=='J' & clarity=='VS1') %>% log_plot("carat", "price", "cut")
```

## purrr

- `map`: lists
- `pmap`: rows of dataframe
- `map2`: combine two lists (or df rows)
- `imap`: short hand for map2(x, names(x), ...) if x has names, or map2(x, seq_along(x), ...) if not

- group and summarise with function that includes multiple outputs
```{r}

# long
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = list(quantile(Petal.Length, c(0.25, 0.5, 0.75)))) %>%
  mutate(pl_qtile = map(pl_qtile, enframe, name = "quantile")) %>%
  unnest() %>%
  mutate(quantile = factor(quantile))
  
# wide 
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = list(quantile(Petal.Length, c(0.25, 0.5, 0.75)))) %>%
  mutate(`25%` = map_dbl(pl_qtile, 1),
         `50%` = map_dbl(pl_qtile, 2),
         `75%` = map_dbl(pl_qtile, 3)) %>%
  dplyr::select(-pl_qtile)

# can also use spread
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = list(quantile(Petal.Length, c(0.25, 0.5, 0.75)))) %>%
  mutate(pl_qtile = map(pl_qtile, enframe)) %>%
  unnest() %>%
  spread(name, value)
```

## transpose a data frame
- `t()` works for matricies, but it's a pain to move ids back and forth between row names and columns
- this is much nicer

```{r}
df %>% gather(id, value, -Sample) %>% spread(Sample, value)

```