---
layout: single
title: "Demonstration of `tidytext` using Darwin's \"On the Origin of Species\"."
date: 2017-04-17
type: post
published: true
categories: ["Hacking"]
tags: ["R", "Demonstration", "tidytext"]
excerpt: "A short demonstration of the R package tidytext using the 1st edition of 'the Origin of Species'."
---



> This post is an extended version of what I put together for
> the
> [Gainesville R User Group](https://www.meetup.com/R-Users-Group-Gainesville-FL/) meetup
> of April 4th, 2017. The Rmd file for this post is on [GitHub](https://github.com/fmichonneau/francoismichonneau.net/blob/gh-pages/_posts/2017-04-17-tidytext-origins-of-species.Rmd)

We are going to use the "Origin of Species" (1st edition, published in 1859) to explore some of the functionalities of the `tidytext` package. Most of the code used here is coming from the book written by the authors of the package, Julia Silge & David Robinson: "[Text Mining with R](http://tidytextmining.com/)". I encourage you to read the book if you want to learn more about this topic. It's really clear and entertaining to read!


## First things first


Let's load the packages that we'll need for this demonstration:


```r
library(tidytext)
library(dplyr)
library(readr)
library(ggplot2)
library(tidyr)
library(stringr)
library(purrr)
library(hrbrthemes)
```


## Load in the text in memory

To load the text of the book, we need to use the GitHub version from the `gutenbergr` package. The version on CRAN uses a download mirror that is currently not working, the version of GitHub uses a different mirror to address this problem.

You can use the `install_github` function from either the `devtools` or `remotes` packages to download and install this development version of the package from GitHub:


```r
## remotes::install_github("ropenscilabs/gutenbergr")
library(gutenbergr)
```

Let's find the "Origin" in the list of books made available by the Gutenberg Project, by using `str_detect` from `stringr` to find potential matches


```r
res <- gutenberg_works(str_detect(title, regex("on the origin of species", ignore_case = TRUE)))
res %>% select(title)
```

```
## # A tibble: 3 × 1
##                                                                           title
##                                                                           <chr>
## 1 On the Origin of Species By Means of Natural Selection\r\nOr, the Preservatio
## 2   A Critical Examination of the Position of Mr. Darwin's Work, "On the Origin
## 3 On the Origin of Species by Means of Natural Selection\r\nor the Preservation
```

```r
res %>% select(gutenberg_id)
```

```
## # A tibble: 3 × 1
##   gutenberg_id
##          <int>
## 1         1228
## 2         2926
## 3        22764
```

There are 3 books that contains "on the origin of species" in the title. It looks like there is the 1st edition (what we want), a book about "the origin of species", and the 6th edition also by Darwin.

Let's download the 1st edition. To do so, we need to provide the `gutenberg_id` to the `gutenberg_download` function:


```r
ofs_full <- gutenberg_download(1228)
```

You get the entire book in a data frame in less time it takes to get a sip of tea!

## Make it tidy

In his young age, while traveling on the HMS Beagle, Darwin was apparently not very tidy. He didn't label the finches he collected in the Galapagos archipelago by island. However, with the help of an ornithologist, and other specimens that were collected at a different time and correctly labeled by islands, he managed to figure out where each bird had been collected.

This is no excuse to not make the text tidy...

We are going to do a few things to it:

1. Remove the preface
1. Remove the table of contents
1. Remove the index
1. Figure out where the chapters are, so we can label each line with the chapter it's coming from
1. Remove the blank lines
1. Add line numbers

### Remove the preface

If we look at the original text file for the book, we see that the text does not start until after the cover page, the forewords, and the table of contents. The book only start with the "Introduction" chapter.

Here we are going to use the `slice` function from `dplyr` to extract the lines in the data frame to only retain the text of the book. So, we are using the `grep` function to return the row number where the word introduction occurs:


```r
grep("INTRODUCTION\\.", ofs_full$text)
```

```
## [1]  49 275
```

It occurs in 2 places: once in the table of contents, and once as the title of the introductory chapter. There are a couple of white spaces in front of the word in the table of contents though so we can modify our regular expression by adding a `^` in front of it to specify the line has to exactly start with the word "INTRODUCTION":


```r
grep("^INTRODUCTION\\.", ofs_full$text)
```

```
## [1] 275
```

We get a single match, and where we want it.

### Table of contents

The table of contents starts with the line "INDEX.". Let's use the same approach to find it:


```r
grep("^INDEX\\.", ofs_full$text)
```

```
## [1] 14214
```

Again we get a single match, and where we expect it to be. So now we know how to extract the boundaries of the actual text for the book.


```r
ofs <- ofs_full %>%
  slice(grep("^INTRODUCTION\\.", text):(grep("^INDEX\\.", text))-1)
```



### Chapter limits detection

It looks like each chapter starts with a number, followed by a period, followed by a fully capitalized title. For instance: "1. VARIATION UNDER DOMESTICATION". Let's see how we can do that...

Let's start by only matching the lines that begin with a number followed by a
period and see how we fare:


```r
grep("^[0-9]+\\.", ofs$text, value=TRUE)
```

```
##  [1] "1. VARIATION UNDER DOMESTICATION."                   
##  [2] "2. VARIATION UNDER NATURE."                          
##  [3] "3. STRUGGLE FOR EXISTENCE."                          
##  [4] "4."                                                  
##  [5] "5. LAWS OF VARIATION."                               
##  [6] "6. DIFFICULTIES ON THEORY."                          
##  [7] "7. INSTINCT."                                        
##  [8] "8. HYBRIDISM."                                       
##  [9] "9. ON THE IMPERFECTION OF THE GEOLOGICAL RECORD."    
## [10] "10. ON THE GEOLOGICAL SUCCESSION OF ORGANIC BEINGS." 
## [11] "11. GEOGRAPHICAL DISTRIBUTION."                      
## [12] "12. GEOGRAPHICAL DISTRIBUTION--continued."           
## [13] "13. MUTUAL AFFINITIES OF ORGANIC BEINGS: MORPHOLOGY:"
## [14] "14. RECAPITULATION AND CONCLUSION."
```

It looks like we get all the chapter boundaries almost correctly. The only hiccup is for Chapter 4 where the title is on a different line from the number. For the purpose of this demonstration, we are going to leave it as is.

We could also add a match for the introduction, but I'm going to leave it as it
is, so the introduction will be labeled 0, and the other chapters will have the
numbers as the ones Darwin gave them.

### Blank lines

To remove the blank lines in our data frame, we are using the function `nzchar` (non-zero character) that returns `TRUE` if a string is not blank. To give you an idea of how it works:


```r
head(nzchar(ofs$text))
```

```
## [1] FALSE  TRUE FALSE  TRUE  TRUE  TRUE
```

### Putting it all together

Now that we have all the steps, we can pipe them together, and create new columns to keep track of the line number and chapters. We use replace `grep` with `grepl` to get `TRUE` for the lines that match the chapter boundaries, which combined with `cumsum` will be incremented to reflect the chapter number (using `sum` on logical vectors is probably one of my favorites R trick).


```r
ofs <- ofs_full %>%
  slice(grep("^INTRODUCTION\\.", text):(grep("^INDEX\\.", text))-1) %>%
  filter(nzchar(text)) %>%
  mutate(linenumber = row_number(),
         chapter = cumsum(grepl("^[0-9]+\\.", text)))
```

### Using tidytext to make it tidy

Now that we have everything in order, we can use the `tidytext` package to make the text ready for analysis. Each word found in the text will be converted to lowercase, the punctuation will be removed, and we will have the line number and the chapter for each occurence of the word. All of this is taken care by the `unnest_tokens` function from `tidytext`:


```r
ofs_tidy <- ofs %>%
  unnest_tokens(word, text)
```

The final step before analysis is the removal of the "stop words" using the magic of an `anti_join` (from `dplyr`):


```r
data("stop_words")

ofs_tidy <- ofs_tidy %>%
  anti_join(stop_words)
```

```
## Joining, by = "word"
```

We can now start analyzing the text, and asking some real questions!

## What are the most common words in the Origin of Species?

I'll give you a clue... it's in the title...


```r
ofs_tidy %>%
  count(word, sort=TRUE) %>%
  top_n(15) %>%
  mutate(word = reorder(word, n)) %>%
  ggplot(aes(x = word, y = n)) +
  geom_col() +
  xlab(NULL) +
  coord_flip() +
  theme_ipsum_rc()
```

```
## Selecting by n
```

![plot of chunk most-common-words]({{ base_path }}/images/2017-04-17-tidytext-most-common-words-1.svg)

Yes, it was "species"! The other common words are also very interesting. I can't help but wonder if Darwin counted the words himself to make sure he used "forms" and "varieties" the same number of times (397 and 396 respectively...). He also made sure that he gave the same attention to "life", "plants" and "animals". No silly distinction between botanists and zoologists, his ideas applied to all domains of life. He also made it clear that apparently, all the selection he's talking about in the book is natural. Let's check!


## Relationships between words

Let's examine the bigrams (2-word combinations) in the book. Here we will tokenize the text into pairs of 2 consecutive words, and remove the occurences that contain stop words:


```r
ofs_bigram <- ofs %>%
  unnest_tokens(bigram, text, token="ngrams", n=2)

ofs_separated <- ofs_bigram %>%
  separate(bigram, c("word1", "word2"), sep =" ")

ofs_filtered <- ofs_separated %>%
  filter(!word1 %in% stop_words$word,
         !word2 %in% stop_words$word) %>%
  unite(bigram, word1, word2, sep = " ")
```

Let's count and plot:


```r
ofs_filtered %>%
  count(bigram, sort=TRUE) %>%
  top_n(15) %>%
  mutate(bigram = reorder(bigram, n)) %>%
  ggplot(aes(x=bigram, y=n)) +
  geom_col() +
  coord_flip() +
  theme_ipsum_rc()
```

```
## Selecting by n
```

![plot of chunk plot-bigrams]({{ base_path }}/images/2017-04-17-tidytext-plot-bigrams-1.svg)

Yes, if "natural" and "selection" occur roughly the same number of time, it's not a coincidence! It's _by far_ the most common bigram in the book!

I don't to over interpret this, but it seems that it shows other interesting patterns:

- he uses "closely allied" and "allied species" to emphasize the importance of looking at closely related species to understand natural selection
- the role of inheritence is shown by the common terms "modified descendants", "common parents", and "parent species"
- his observations in South (and North) America really shaped his thinking
- he emphasizes the role of "physical conditions" and "glaciation"
- he uses "oceanic islands" and "fresh water" bodies as natural laboratories for natural selection
- and obviously, "domestic animals" including the "rock pigeon" make it to the top 15 of the most common bigrams.

## Sentiment analysis

The `tidytext` package comes with 3 lexicon that classify common English words as being associated with negative or positive feelings. Their scoring system vary, they are based on single words (no sense of context), and they have been established much more recently than 1859. So doing a sentiment analysis using these lexicons on "The Origin of Species" may not be very accurate.

The code below standardizes the lexicons to only get whether a word is positive and negative and is averaged over groups of 80 lines. If you want more details, go read Julia and David's book because they explain these different steps much better than I could dream of:



```r
afinn <- ofs_tidy %>%
  inner_join(get_sentiments("afinn")) %>%
  group_by(chapter, index = linenumber %/% 80) %>%
  summarize(sentiment = sum(score)) %>%
  mutate(method = "AFINN")
```

```
## Joining, by = "word"
```

```r
bing_and_nrc <- bind_rows(ofs_tidy %>%
                            inner_join(get_sentiments("bing")) %>%
                            mutate(method = "Bing et al."),
                          ofs_tidy %>%
                            inner_join(get_sentiments("nrc") %>%
                                         filter(sentiment %in% c("positive", "negative"))) %>%
                            mutate(method = "NRC")) %>%
  count(method, index = linenumber %/% 80,  chapter, sentiment) %>%
  spread(sentiment, n, fill = 0) %>%
  mutate(sentiment = positive - negative)
```

```
## Joining, by = "word"
## Joining, by = "word"
```

```r
bind_rows(afinn, bing_and_nrc) %>%
  ggplot(aes(index, sentiment, fill = as.factor(chapter))) +
  geom_col() +
  facet_wrap(~ method, ncol = 1, scales = "free_y") +
  theme_ipsum_rc() +
  scale_fill_discrete(name="", labels=c("Introduction",
                                        paste("Chapter", 1:14)))
```

![plot of chunk sentiment]({{ base_path }}/images/2017-04-17-tidytext-sentiment-1.svg)

I notice a few things:

- NRC seems to have much higher positivity scores than the 2 other lexicons.
- Both AFINN and Bing et al. show strong negativity for chapter 3. Its title you ask? "Struggle for existence". That sounds about right.


## Features of the final chapter

To finish this rapid analysis of "The Origin of Species", let's look at the most distinctive words in the conclusion compared to the rest of the book. For this, we'll use the log odds ratio for each word that occur at least 10 times. We'll select the 15 most distinctive words from the entire book compared to the final chapter.



```r
word_ratios <- ofs_tidy %>%
    group_by(conclusion = chapter == 14) %>%
    count(word, conclusion) %>%
    filter(n >= 10) %>%
    ungroup() %>%
    spread(conclusion, n, fill = 0) %>%
    rename(conclusion = `TRUE`, restofbook = `FALSE`) %>%
    mutate_if(is.numeric, funs((. + 1)/sum(. +1))) %>%
    mutate(logratio = log(conclusion/restofbook)) %>%
    arrange(desc(logratio))

word_ratios %>%
    mutate(abslogratio = abs(logratio)) %>%
    group_by(logratio < 0) %>%
    top_n(15, abslogratio) %>%
    ungroup() %>%
    mutate(word = reorder(word, logratio)) %>%
    ggplot(aes(word, logratio, fill = logratio < 0)) +
    geom_col() +
    coord_flip() +
    ylab("log odds (Chapter 14/Rest of Book)") +
    scale_fill_discrete(name = "", labels = c("Chapter 14", "Rest of Book")) +
    theme_ipsum_rc()
```

![plot of chunk compare-conclusion]({{ base_path }}/images/2017-04-17-tidytext-compare-conclusion-1.svg)

For this, it seems clear that the words chosen by Darwin in the conclusion are more abstract ("theory", "laws", "view") than in the rest of the book. The multiple instances of "created" and "creation" in this final chapter are all used to refute creationism, for instance (emphasis mine):

> Several eminent naturalists have of late published their belief that a
> multitude of reputed species in each genus are not real species; but that
> other species are real, that is, have been independently **created**.  This
> seems to me a strange conclusion to arrive at. They admit that a multitude of
> forms, which till lately they themselves thought were special **creations**

To finish, a famous characteristic of the book is that the word evolution does not appear in it. However, the book ends with the word "evolved". Let's double check, by looking for words that starts with "evol"


```r
ofs_tidy %>%
    filter(str_detect(word, "^evol"))
```

```
## # A tibble: 1 × 4
##   gutenberg_id linenumber chapter    word
##          <int>      <int>   <int>   <chr>
## 1         1228      13138      14 evolved
```

Indeed, the word "evolved" only occurs once in the book, and in the Chapter 14. And we can verify that it's the last line:


```r
max(ofs_tidy$linenumber)
```

```
## [1] 13138
```

This was a short text analysis on the Origin of Species. There is a lot more that I would like to do, but that it will be for another day.

This short demonstration really exemplifies how powerful the tidy format is. By weaving together different packages in this ecosystem, the barrier of entry for using a new package (I hadn't used tidytext before this) is low, and you can focus on your analysis rather than having to worry about data structures.


#### Session info


```r
sessionInfo()
```

```
## R version 3.3.3 (2017-03-06)
## Platform: x86_64-pc-linux-gnu (64-bit)
## Running under: Ubuntu 16.10
## 
## locale:
##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
##  [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
##  [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
##  [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
## [11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
## 
## attached base packages:
## [1] stats     graphics  grDevices datasets  utils     methods   base     
## 
## other attached packages:
##  [1] gutenbergr_0.1.2.9000 hrbrthemes_0.1.0      stringr_1.2.0        
##  [4] dplyr_0.5.0           purrr_0.2.2           readr_1.1.0          
##  [7] tidyr_0.6.1           tibble_1.3.0          ggplot2_2.2.1        
## [10] tidyverse_1.1.1       tidytext_0.1.2        devtools_1.12.0      
## [13] BiocInstaller_1.24.0 
## 
## loaded via a namespace (and not attached):
##  [1] Rcpp_0.12.10      lubridate_1.6.0   lattice_0.20-35  
##  [4] clisymbols_1.1.0  assertthat_0.2.0  digest_0.6.12    
##  [7] psych_1.7.3.21    R6_2.2.0          plyr_1.8.4       
## [10] evaluate_0.10     httr_1.2.1.9000   highr_0.6        
## [13] lazyeval_0.2.0    curl_2.4          readxl_0.1.1     
## [16] rstudioapi_0.6    extrafontdb_1.0   Matrix_1.2-8     
## [19] urltools_1.6.0    labeling_0.3      extrafont_0.17   
## [22] selectr_0.3-1     foreign_0.8-67    triebeard_0.3.0  
## [25] munsell_0.4.3     hunspell_2.3      broom_0.4.2      
## [28] compiler_3.3.3    janeaustenr_0.1.4 modelr_0.1.0     
## [31] mnormt_1.5-5      notifier_1.0.0    XML_3.98-1.6     
## [34] crayon_1.3.2      withr_1.0.2       SnowballC_0.5.1  
## [37] grid_3.3.3        nlme_3.1-131      jsonlite_1.3     
## [40] Rttf2pt1_1.3.4    gtable_0.2.0      DBI_0.6          
## [43] git2r_0.18.0      magrittr_1.5      scales_0.4.1     
## [46] tokenizers_0.1.4  stringi_1.1.3     foghorn_0.4.2    
## [49] reshape2_1.4.2    xml2_1.1.1        fortunes_1.5-4   
## [52] tools_3.3.3       forcats_0.2.0     hms_0.3          
## [55] parallel_3.3.3    colorspace_1.3-2  rvest_0.3.2      
## [58] memoise_1.0.0     knitr_1.15.1      haven_1.0.0
```
