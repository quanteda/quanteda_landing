---
title: "Comparing the performance of quanteda with other R packages"
author: "Kohei Watanabe and Stefan Müller"
date: "2018-05-02"
---



One of the most frequent questions about the **quanteda** (version 1.2.1) package concerns its performance compared to other R packages for quantitative text analysis. In this page, we compare **quanteda** with the popular [**tm**](https://cran.r-project.org/package=tm) (version 0.7.3) and [**tidytext**](tidytext package) (version 0.1.8) packages in terms of execution time. We focus on three common operations of text analysis that we can accomplish by all three packages: tokenization, feature selection, and document-feature matrix (or document-term matrix) construction. 

We measure the execution time with the **microbenchmark**. We repeat each operation 20 times to obtain distribution of execution time. The benchmarking code is [available in the website repository](https://github.com/quanteda/quanteda_landing/tree/master/content/performance.Rmarkdown).

## 1. Preparation

**quanteda** supports multi-threading but the number of threads defaults to two due to CRAN's regulation, so users have to set the value through `quanteda.options()` to maximize its performance.


```r
quanteda_options("threads" = 8)
```

We use a corpus from 6000 articles published in the *Guardian* for benchmarking. The corpus consists of over 5.3 million tokens and is part of [**quanteda.corpora**](https://github.com/quanteda/quanteda.corpora).
 

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

# create a data frame with document identifier
corp_ti <- data_frame(txt = txt, document = seq_along(txt))
```

## 2. Tokenization

Having created the corpus objects for each package, we measure the time it takes to tokenize the corpus. **quanted** and **tm** have multiple tokenizers, but these are the fastest options.


```r
times_token <- microbenchmark(
    tm = tm::Boost_tokenizer(corp_tm),
    quanteda = quanteda::tokens(corp_qu, what = "fastestword"),
    tidytext = tidytext::unnest_tokens(corp_ti, 'token', 'txt', to_lower = FALSE),
    times = 20, unit = "s"
)
```

**quanteda**'s and **tidytext**'s execution times are very similar as both packages rely on the **stringi** package for tokenization. The operation takes much longer in **tm** than in the other two packages.


```
##       expr   min    lq  mean median    uq   max neval
## 1       tm 13.60 13.93 14.09  14.00 14.26 14.77    20
## 2 quanteda  1.02  1.25  1.30   1.34  1.37  1.47    20
## 3 tidytext  1.03  1.11  1.19   1.19  1.29  1.34    20
```

<img src="/performance_files/figure-html/unnamed-chunk-7-1.png" width="576" />

## 3. Feature selection

Here, we remove grammatical words from the texts to measure time in feature selection. We use the list of 175 English grammatical words from the [**stopwords**](https://github.com/quanteda/stopwords) package.


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

**quanteda** outperform **tidytext** and **tm** in feature selection.


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

In terms of transforming tokenized text to a document-feature matrix, the execution times for **tm** and **quanteda** are similar. **tidytext** is slower because it involves an intermediate step (counting the occurrences of each term) before we transform it to **quanteda**'s `dfm` object.


```
##       expr   min    lq  mean median    uq   max neval
## 1       tm  9.68 10.07 10.40   10.3 10.63 11.45    20
## 2 tidytext 16.20 16.57 17.30   17.1 17.55 20.39    20
## 3 quanteda  4.30  4.64  4.93    4.8  5.12  6.29    20
```

<img src="/performance_files/figure-html/unnamed-chunk-12-1.png" width="576" />

## 5. Conclusion

Overall, we see that the biggest advantage of **quanteda** and **tidytext** compared to the **tm** package lies in the much fast tokenization of texts. **quanteda** and **tidytext** perform similarly in  tokenization due to the common underlying package. However, **quanteda** is much faster than **tidytext** in both feature selection and document-feature matrix construction.

