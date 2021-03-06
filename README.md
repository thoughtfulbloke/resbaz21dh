---
title: "Text Analysis Workshop"
author: "David Hood"
date: "02/06/2021"
output:
  html_document: 
    keep_md: yes
  word_document: default
---



This is the tutorial document for the session "Text Analysis in R" being run by David Hood as part of Research Bazaar 2021: 5-7 July in Dunedin. The intended level is giving people comfortable with the general use of R (and the Tidyverse within R) exposure to some key ideas in text analysis.

People preparing for this workshop should use the download button at the top to download the repo, and open the .Rmd version of the document in R to work with.

This tutorial assumes your machine is set up with R and R Studio.

Recommended follow-on material is the book "Text Mining with R" by Julia Silge and David Robinson available as a website at: https://www.tidytextmining.com

## Preparing to analyse some text

We want these added R packages:

* tidytext, for a lot of introductory text analysis tools
* textdata, for some tidytext helper librarys
* tm, for text manipulation tools such as stemming
* dplyr, for some data manipulation tools
* stringr, for some text handling functions
* ggplot2 for making some graphs of results
* topicmodels for LDA document modelling
* reshape2 for an organsing data dependency
* ggrepel for clearly labelled graphs
* tidyr for consolidating columns

If using a Windows or Macintosh, haven't yet installed these and are not familiar with installing package here is some code that should install prepared versions not needing extra software to install.

````
install.packages("tidytext", type = "binary", dependencies=TRUE)
install.packages("textdata", type = "binary", dependencies=TRUE)
install.packages("tm", type = "binary", dependencies=TRUE)
install.packages("dplyr", type = "binary", dependencies=TRUE)
install.packages("stringr", type = "binary", dependencies=TRUE)
install.packages("ggplot2", type = "binary", dependencies=TRUE)
install.packages("topicmodels", type = "binary", dependencies=TRUE)
install.packages("reshape2", type = "binary", dependencies=TRUE)
install.packages("ggrepel", type = "binary", dependencies=TRUE)
install.packages("tidyr", type = "binary", dependencies=TRUE)
````

Ahead of the session it is also a good idea to download the nrc sentiment analysis corpus we will be using. Run these two lines of code in the console, and then type 1 for Yes (Yes to download the corpus). This saves having to download the corpus during the training session.

```
library(textdata)
nrc <- lexicon_nrc()
```

During the session, we need to load the relevant packages for the session.


```r
library(dplyr)
library(stringr)
library(tidytext)
library(tm)
library(textdata)
library(topicmodels)
library(ggplot2)
library(ggrepel)
library(tidyr)
```


Also, getting text to analyse from different formats of sources (pdfs, Word docs, web pages) may involve specialist packages for that specific kind of source material.

For this tutorial, we are using a text file of the 1891 public domain book "With Axe and Rope in the New Zealand Alps", by George Edward Mannering, available from Project Gutenburg at https://www.gutenberg.org/ebooks/60919

As a text file, it could be read in as one document then divided up into sections as needed (such as sentences), or read in at another level such as lines (which equates to paragraphs).

For this tutorial we will read the document in as a series of lines.


```r
file_contents <- readLines("Mannering_1891.txt")
```

Reading the file, because understanding the structure of what is being analysed is helpful, we might identify features we need to tidy up so the file meets our assumptions about organisation.

For example, the electronic version of the book has a line break at the end of each line, rather than (like a Word document) a line break at the end of each paragraph, with paragraphs separated by a blank line. We can confirm that there are a lot of blank lines (rather than lines with spaces in them) by checking how many characters each line has.

```r
nchar(file_contents)[1:10]
```

We could try strategies of joining text back together, or of marking out which paragraphs the lines of text belong to. In this case we will mark each line by the original order and which paragraph it belongs to.

As we are starting to create annotations and structure with our data, we will turn it from a vector of text, into a spreadsheet arrangement containing a column for the text, and other columns for specific annotations we are tracking.


```r
mannering <- data.frame(text = file_contents,
                        stringsAsFactors = FALSE)
```

Line numbers can be added for posterity with a row_number() command from dplyr. Paragraphs can be recorded by testing if the the line is blank, then taking advantage of turning TRUE & FALSE test results of empty lines as 1 & 0 numbers to cumulatively sum the lines into paragraph groups.


```r
mannering <- mannering %>%
  mutate(line_n = row_number(),
         para_n = cumsum(nchar(text) == 0))
```

As well as lines and paragraphs, this book also contains chapters. If we want to approach the analysis in a "per chapter" way, we need to mark what chapters lines are in. From reading the contents, Chapters do start with the word chapter, but that may not be enough by itself as other unwanted lines may include the word chapter.

We can check the occurrences of chapter in context with a grep() search command


```r
grep("chapter", mannering$text, value=TRUE, ignore.case = TRUE)
```

Group Discussion: *What can we tell about the use of the term "Chapter", how might we leverage the term, and are there any other terms that are used for chapter like units?*

In implementing this in a similar way to the paragraphs, the function str_starts() from stringr might be helpful to use. Multiple different possible starts could be separated using the OR symbol | to mark out looking for different terms.


```r
# try extending your own code after the or (| symbol) for 
# splitting chapters by the start if kubes
# for those who need it, there is an example solution for the chapter
# splitting exercise on at the end of the document

mannering <- mannering %>%
  mutate(chapter_n = cumsum(str_starts(text, fixed("CHAPTER")) |
                               # replace the TRUE with your code
                              TRUE
                              ))
```

As well as aggregating the input data to mark out larger units of meaning, we also often need to break the data down into individual semantic components. This is normally done by sending the data through a function that can pick out words, with the nuances being around what counts as a word/semantic unit (such as are emojis words, or are web addresses with dots and slashes between the parts one unit or several).

In this case it is a english word form text and we will use traditional english rules in tokenising the text lines into word tokens.


```r
mannering_words <- mannering %>%
  unnest_tokens(output=word, input = text, token = "words") %>%
  mutate(word_n = row_number())
```

Discussion: *In aggregating up from lines to paragraphs and chapters, and down to words, what unit of analysis might have been missed?*

We aren't going back and changing things for this question, but it is an example of how treating the data differently at an earlier stage would have created different opportunities.

## Counting things up

Sometimes, you just want to count things up


```r
mannering_words %>%
  count(word) %>%
  filter(word == "aorangi") %>%
  ungroup()
```

If you want a count by semantic unit, a common practice is to group the data by that semantic unit and then count within it


```r
mannering_words %>%
  group_by(chapter_n) %>%
  count(word) %>%
  filter(word == "aorangi") %>%
  ungroup()
```
If you wanted the words as a percentage of the words in the chapter, you might divide the total for a word by the number of words in the chapter.


```r
mannering_words %>%
  group_by(chapter_n) %>%
  count(word) %>%
  mutate(percentage = 100 * n / sum(n)) %>%
  filter(word == "aorangi") %>%
  ungroup()
```
Because this method is counting words that are there, rather than words that are not, it is leaving out chapters 9 and 11. So another approach is counting the number of words that match the word we are looking for (technically counting test results rather than counting words as items)


```r
mannering_words %>%
  group_by(chapter_n) %>%
  summarise(percentage_aorangi = 100 * sum(word == "aorangi") /n()) %>%
  ungroup()
```
If we wanted to compare frequency of Aorangi with Mount Cook, we would need to know not just the frequency Mount, but the frequency of Mount followed by Cook (or Cook preceded by Mount).

A lot of visualizations of count data are ultimately bar graphs.


```r
# copy the preceeding code and have the data flow into the graph

  ggplot(aes(x=chapter_n,y=percentage_aorangi, fill=chapter_n)) + 
  geom_col() + coord_flip() + theme_minimal()
```

Discussion: *How could this graph be improved*

For working with preceding or following words, the dplyr package has the lead() and lag() functions which act to shift rows up (lead) or down (lag) the data. There is a second, optional, setting of the number of rows to shift up or down (the default is one).

Using lag or lead is normally done within a semantic unit such as a sentence, as it makes little sense to imply a robust connection between the last word in one sentence and the first word of a following sentence, even though the words follow in order. For this tutorial, as we did not record sentence information about the data (see earlier discussion about limitations) we will use paragraphs instead to create a new column called "following" that uses lead to record the following word for each paragraph


```r
mannering_words <- mannering_words %>%
  group_by(para_n) %>% 
  mutate(following=lead(word)) %>%
  ungroup()
```

As an exercise, try taking the earlier aorangi counts, and calculating an additional column of mount_cook for use of Mount Cook. To run a logical test on more than one column, the and operator & may be helpful.


```r
mannering_words %>%
  summarise(aorangi = sum(word == "aorangi"),
            mount_cook = sum(word == "mount" & following == "cook")) 
# based on the above way of calculating totals try and combine 
# this with the above code snippets for searches for aorangi
# to develop code that tells the percentage for both in each chapter
# for a subsequent exercise to work OK the two summary variables
# should be named percentage_aorangi and percentage_mount_cook
```

For showing change across a range for a limited set of things, a slope graph is a possible choice (which is in reality a line graph)


```r
# take the percentage aorangi vs mount cook and flow it into a slope graph.
# as aorangi and mount_cook are columns, we need to combine them before graphing
mannering_words %>%
  group_by(chapter_n) %>%
  summarise(percentage_aorangi = 100 * sum(word == "aorangi") /n(),
            percentage_mount_cook = 100 * 
              sum(word == "mount" & following == "cook") /n()) %>%
  ungroup() %>%
  gather(key = terms, value=percentage, percentage_aorangi:percentage_mount_cook) %>% 
  ggplot() +
  geom_vline(aes(xintercept=chapter_n), colour="#444444") +
  geom_line(aes(x=chapter_n, y=percentage, colour=terms, linetype=terms)) + 
  geom_point(aes(x=chapter_n, y=percentage, colour=terms)) +
  theme_minimal()
```

If wanting to fuse together particular terms, such and Mount and Cook, so that they are treated as a single "word" (possibly by using a case_when statement creating a new term for certain word + following word combinations and using the original terms for the rest), you might want a report on common pairs in the document.

If you have two (or more) columns, you can count combinations inside a count() function by listing all the column names, and there is an additional sort argument to list the commonest first.


```r
mannering_words %>% count(word, following, sort=TRUE)
```

Discussion: *If you write some code and look through the results, what is the main problem with this as a source of pairs data:*

Try to determine counts of the words prior to the word "peaks" 


```r
mannering_words %>% 
  # your code goes here
  count(word, following, sort=TRUE)
```

## Stop words

In text analysis Stop Words is the term for common words not of interest to the analysis. One strategy is to remove common generic words before making comparisons in frequency, for instance by a dplyr database style anti_join() to include only those terms not in the list of common "stop word" terms.

However, removing stop words can effect things depending on when you remove them in the process- If you remove a bunch of words then look for the "next" word, it may not be the same next word as with all the data.


```r
mannering_words %>%
  anti_join(stop_words, by="word") %>%
  group_by(para_n) %>%
  mutate(stopped_following = lead(word)) %>%
  ungroup() %>%
  filter(following != stopped_following) %>%
  count(word, following, stopped_following, sort=TRUE)
```

## Stemming

Climbers climb, and while they do so they are climbing. For the purposes of a given analysis, does it matter exactly which form of the word was used, or does it just matter that climbs were involved.

One approach is to stem terms into the shortest common form, by using a function that understands a languages use of suffixes. In this case we will draw on the tm package, and the stemDocument function. 

Try taking the starting code and counting the three commonest (stemmed) words per chapter.


```r
mannering_words %>%
  mutate(stemmed = stemDocument(word, language = "english")) %>%
  # your code goes here
  slice(1:3) %>%
  ungroup()
```

## Sentiment analysis

Stop words was an example of a applying a list of words that someone has created to help your analysis. Sentiment analysis is very similar- someone has created a list of words and assigned sentiments to those words. Sentiment Analysis is combining those classified words with the data being analysed. Sets of sentiment analysis words can provide their categorisation in different ways depending on the source. Some provide a positive/negative score, some provide categories of sentiment.

For this example we are using the nrc sentiment corpus, in which words have been labelled with sentiment categories. We apply the labels and count up the use.


```r
nrc <- lexicon_nrc()
mannering_words %>%
  inner_join(nrc) %>%
  count(chapter_n,sentiment, sort=TRUE)
```

Because an inner_join is "for words that appear in both the data and the sentiment lexicon" the results can be very sensitive to the degree of matching between the words in the data and the lexicon


```r
# try the previous code but with a left_join rather than an inner_join
```

Discussion: *What strategies might be used to change the amount of unmatched material? When might they be needed? What are the consequences of those strategies.*

## Many words, many documents.

If comparing several documents, sometimes it is not just how often words appear (term frequency), of interest is words that are more common when compared to other documents. A frequently used method for doing this is TF-IDF (Term Frequncy - Inverse Document Frequency).


```r
mannering_words %>%
  count(chapter_n, word) %>%
  bind_tf_idf(word, chapter_n, n) %>% 
  group_by(chapter_n) %>%
  slice_max(tf_idf, n=10)
```

Discussion: *What does focusing on the distinctively different language hide or show?*

A common format for holding the data in when doing comparisons between documents is a Document Term Matrix (DTM) which counts word use in a format of one row per word and one column per document. The tidytext package has a cast_dtm() function for turning data with document, word, and count columns into this format (and a tidy() function for turning it back). This document term matrix can then be used by functions that expect data in this specialist format.

One such special function in LDA (Latent Dirichlet allocation), which assumes topics have words in common, and documents have topics in common, then (for a given number of limited topics specified by you) tries to find the relationship between topics and documents.



```r
mannering_dtm <- mannering_words %>%
  count(chapter_n, word) %>%
  cast_dtm(chapter_n, word, n)

mannering_lda <- LDA(mannering_dtm, k = 2)
chapter_topics <- tidy(mannering_lda, matrix = "gamma")
```

Because enforcing 2 topics gives us two dimensions, we can plot our data on that basis, but we need to rearrange the data so we have a column for each dimension we want to plot along


```r
chapter_topics %>%
  mutate(topic = paste0("topic_",topic)) %>%
  spread(key=topic, value=gamma, fill = 0) %>%
  ggplot(aes(x=topic_1, y=topic_2, label=document))+
  xlim(-.25,1.25) + ylim(-.25,1.25) +
  geom_point(size=2) + geom_text_repel(max.overlaps = 20, box.padding = 5) +
  theme_void()
```


## Example Solutions

### A chapter splitting solution


```r
mannering <- mannering %>%
  mutate(chapter_n = cumsum(str_starts(text, fixed("CHAPTER")) |
                               str_starts(text, fixed("L???ENVOI")) |
                               str_starts(text, fixed("APPENDIX")) |
                               str_starts(text, fixed("_A SHORT GLOSSARY")) |
                               str_starts(text, fixed("Transcriber???s note"))))
```

### Percentage of Mount Cook use by chapter


```r
mannering_words %>%
  group_by(chapter_n) %>%
  summarise(percentage_aorangi = 100 * sum(word == "aorangi") /n(),
            percentage_mount_cook = 100 * 
              sum(word == "mount" & following == "cook") /n()) %>%
  ungroup()
```


### Words before peak solution


```r
mannering_words %>% 
  filter(following == "peaks") %>%
  count(word, following, sort=TRUE)
```

### example solution 3 commonest stemmed words per chapter


```r
mannering_words %>%
  mutate(stemmed = stemDocument(word, language = "english")) %>%
  group_by(chapter_n) %>%
  count(stemmed, sort=TRUE) %>%
  slice(1:3) %>%
  ungroup()
```



