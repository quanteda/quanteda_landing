---
title: "Comparing the performance of quanteda with other R packages"
author: "Kohei Watanabe, Stefan Müller, and Kenneth Benoit"
date: "2018-04-10"
---




One of the most frequent questions about the **quanteda** (version 1.2.1) package concerns the performance and execution time compared with other packages for quantitative text analysis. In this short blog post, we compare **quanteda** with the popular [**tm**](https://cran.r-project.org/package=tm) (version 0.7.3) and [**tidytext**](tidytext package) (version 0.1.8) packages. We focus on three major steps of text analysis that can be accomplished in all three packages: tokenizing text, removing certain features from a corpus/tokens object, and creating a document-feature matrix (also called document-term matrix). 

We measure the execution time with the **microbenchmark** package repeated each step 20 times for each package and creating summary statistics.

**quanteda** supports multi-threading but it defaults to only two threads due to CRAN's regulation, so you have to set the value through `quanteda.options()`.


```r
quanteda_options("threads" = 8)
```


## 1. Create a corpus

First, we create a corpus from 6000 articles published in the Guardian. The corpus consists of over 5.3 million tokens and is part of **quanteda.corpora**, a package with several text corpora.
 

```r
txt <- texts(quanteda.corpora::download('data_corpus_guardian'))
```




```r
length(txt)
```

```
## [1] 6000
```

```r
print(object.size(txt), units = "MB")
```

```
## 26.9 Mb
```


```r
# create a tm corpus
corp_tm <- tm::Corpus(VectorSource(txt))

# create a quanteda corpus
corp_qu <- quanteda::corpus(txt)

# create a data frame with one observation per text for tidytext
corp_ti <- data_frame(txt = txt, document = seq_along(txt))
```

## 2. Tokenization

Having created the corpus objects for each package, we measure the time it takes to tokenize the corpus. We repeat this process 20 times to obtain distribution of exectutio time.


```r
times_token <- microbenchmark(
    tm = tm::Boost_tokenizer(corp_tm),
    quanteda = quanteda::tokens(corp_qu, what = "fastestword"),
    tidytext = tidytext::unnest_tokens(corp_ti, 'token', 'txt', to_lower = FALSE),
    times = 20, unit = "s"
)
```

**quanteda**'s and **tidytext**'s execution times are identical as both packages rely on the **stringi** package for tokenization. The disadvantage in computational performance of the **tm** becomes evident. Tokenization in **tm** takes much longer than in the other two packages.


```
##       expr   min    lq  mean median    uq   max neval
## 1       tm 13.60 13.93 14.09  14.00 14.26 14.77    20
## 2 quanteda  1.02  1.25  1.30   1.34  1.37  1.47    20
## 3 tidytext  1.03  1.11  1.19   1.19  1.29  1.34    20
```

<img src="/performance_files/figure-html/unnamed-chunk-7-1.png" width="576" />

## 3. Feature selection

In a third step, we remove a selection of terms from the tokenized object. This is a common task as many projects involve removing so called stopwords or punctuation. We use the list of English stopwords from the **stopwords** package and measure the time it takes to remove these tokens in each package.


```r
# determine the list of stopwords
stopword <- stopwords::stopwords('en')

# stopwords need to be a data frame for tidytext
stopword_ti <- data_frame(word = stopword)
```


```r
toks_tm <- tm::scan_tokenizer(corp_tm)
toks_ti <- tidytext::unnest_tokens(corp_ti, 'token', 'txt', to_lower = FALSE)
toks_qu <- quanteda::tokens(corp_qu, what = "fastestword")

times_remove <- microbenchmark(
    tm = tm::tm_map(corp_tm, removeWords, stopword),
    tidytext = dplyr::anti_join(toks_ti, stopword_ti, by = c('token' = 'word')),
    quanteda = quanteda::tokens_remove(toks_qu, stopword),
    times = 20, unit = "s"
)
```

Again, **quanteda** and **tidytext** outperform **tm** in feature selection.


```
##       expr   min    lq  mean median    uq   max neval
## 1       tm 0.792 0.797 0.804  0.801 0.806 0.841    20
## 2 tidytext 0.846 0.864 0.889  0.888 0.910 0.950    20
## 3 quanteda 0.557 0.579 0.613  0.610 0.633 0.700    20
```

<img src="/performance_files/figure-html/unnamed-chunk-10-1.png" width="576" />

## 4. Document-feature matrix construction

Finally, we construct a document-feature matrix from a corpus to compare packages' overall performance.


```r
# tidytext requires multiple functions
tidy_dfm <- function(x) {
    x %>% 
    tidytext::unnest_tokens(token, txt, to_lower = FALSE) %>% 
    dplyr::count(token, document) %>% 
    tidytext::cast_dfm(document, token, n)
}

# measure the execution time for creating a dfm
times_dfm <- microbenchmark(
    tm = tm::TermDocumentMatrix(corp_tm, control = list(tokenize = Boost_tokenizer)),
    tidytext = tidy_dfm(corp_ti),
    quanteda = quanteda::dfm(corp_qu, what = "fastestword"),
    times = 20, unit = "s"
)
```

In terms of transforming tokenized text to a document-feature matrix, the execution times for **tm** and **quanteda** are similar. **tidytext** is slower because it involves an intermediate step (counting the occurences of each term) before we transform it to **quanteda**'s `dfm` object.


```
##       expr  min   lq mean median   uq  max neval
## 1       tm 2.89 2.94 3.10   3.02 3.14 4.07    20
## 2 tidytext 7.58 7.80 8.09   8.09 8.26 8.78    20
## 3 quanteda 1.82 2.00 2.34   2.40 2.48 3.16    20
```

<img src="/performance_files/figure-html/unnamed-chunk-12-1.png" width="576" />

## 5. Colclusion

Overall, we see that the biggest advantage of **quanteda** compared to the **tm** package lies in the much fast tokenization of texts. **tidytext** and **quanteda** perform similarly in  tokenization due to the common underlying package. However, **quanteda** is much faster than **tidytext** in both feature selection and document-feature matrix construction.

