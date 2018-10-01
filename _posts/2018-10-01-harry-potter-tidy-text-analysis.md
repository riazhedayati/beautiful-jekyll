---
layout: post
title: Yer a (text mining) Wizard, Harry!
subtitle: A tidy text analysis of the Harry Potter book series
---
![alt text](/img/HP/HP.jpg "Harry Potter")

As a young teenager in the early 2000s, I have very fond memories surrounding the Harry Potter series. I remember holing up in my room and devouring the newest book release every other summer, shedding some tears when I realized that Sirius was actually dead, and lively discussions with my friends on our theories of what we thought would happen in the next book. My wife and I even visited The [Wizarding World of Harry Potter]( https://www.universalorlando.com/web/en/us/universal-orlando-resort/the-wizarding-world-of-harry-potter/hub/index.html) in Orlando as part of our honeymoon. Suffice it to say, I was a big fan.

While I am not as into Harry Potter anymore as I once was, I thought it would still be fun to relive some of that nostalgia and do some [tidy text]( https://www.tidytextmining.com/) analysis of the Harry Potter book series.  


### Reading and cleaning the text
After some searching, I was able to find .txt versions of all 7 books in the Harry Potter series. Using some ugly loops, I was able to wrangle each of the text files into something a bit more manageable, and capture some metadata about the text along the way. 

<pre><code class="language-r line-numbers">library(tidyverse)
library(syuzhet)
library(tidytext)
library(stringr)
library(zoo)
library(SnowballC)

#initialize filenames to read in
filenames <- list.files("raw text")

#write loop to read in all the books into a list
hp <- list()
for (i in seq_along(filenames)){
  hp[[i]] <- read_tsv(paste0("raw text/", filenames[i]), col_names = c("text"), skip = 1) %>% 
    mutate(pageline = str_detect(text, "Page ")) %>%
    mutate(pagenum = as.numeric(str_extract(text, "\\d+"))) %>% 
    mutate(pagenum = ifelse(pageline == FALSE, NA, pagenum)) %>% 
    na.locf(., fromLast = TRUE) %>% 
    filter(pageline == FALSE) %>%
    mutate(authorname = str_extract(text, "Rowling")) %>% 
    filter(is.na(authorname)) %>% 
    select(-c(pageline, authorname)) %>% 
    mutate(linenum = row_number()) %>% 
    mutate(book = i)
}

#combine all books into one dataset
hp_all_books <- data.frame()
for (i in seq_along(filenames)){
  hp_all_books <- bind_rows(hp_all_books, hp[[i]])
}
</code></pre>

Here is the first paragraph of the first book in our initial dataset:

![alt text](/img/HP/HP_not_tidy.JPG "Untidy Data")

## Tidying the data
Next step was to clean up the dataset. I tidied the data to have one word per row, removed common stopwords that added little value (think _be_ & _this_), and stemmed words to their root form (so that _look_ and _looked_ would be considered the same word). Finally, I did a small cosmetic adjustment to the stemmed words, making them more human readable by replacing each stemmed word with its most common unstemmed version.

<pre><code class="language-r line-numbers">#make dataset tidy, remove common stopwords and stem terms
hp_tidy_nostop_raw <- hp_all_books %>%
  unnest_tokens(original_word, text) %>% #make document tidy and lowercase
  anti_join(stop_words, by = c("original_word" = "word")) %>% #remove stop words
  mutate(word2 = gsub("\\W", "", original_word)) %>% #remove apostrophes
  anti_join(stop_words, by = c("word2" = "word")) %>%  #remove stop words again after removing apostrophes 
  mutate(word_stemmed = wordStem(word2)) %>% #stem words
  select(-word2) 

#group stemmed words together and find the mode of the original word (so that the output graphs are not confusing)
calculate_mode <- function(v) { #define mode function
  uniqv <- unique(v)
  uniqv[which.max(tabulate(match(v, uniqv)))]
}
stemmed_to_original <- hp_tidy_nostop_raw %>% 
  group_by(word_stemmed) %>% 
  summarise(word = calculate_mode(original_word))

#merge stemmed word mode with tidy dataset and keep only the mode of the stemmed words
hp_tidy_nostop <- hp_tidy_nostop_raw %>% 
  left_join(stemmed_to_original, by = "word_stemmed") %>% 
  select(word, book, pagenum, linenum)
</code></pre>

Here is the same paragraph from our tidy and clean dataset:

![alt text](/img/HP/HP_tidy.JPG "Tidy Data")



### What’s the (gg)plot?
To begin our analysis, let’s look at a very simple chart of the top 20 most common words in the entire series:

<pre><code class="language-r line-numbers">#define book colors
book_color <- c("#B82040", "#98D9C1", "#936AB9", "#FF9B4F", "#2484F4", "#78AE1A", "#FDB223")

#find most common words across all books 
hp_tidy_nostop %>%
  count(word, sort = TRUE) %>% 
  top_n(20) %>% #find top 20 words
  mutate(word = reorder(word, n)) %>%
  ggplot(aes(word, n)) +
    geom_col(fill = "#632437") + 
    coord_flip() +
    theme_bw() +
    labs(
        title = "Top 20 Most Common Words",
        subtitle = "All Harry Potter Books", 
        y="Count",
        x=NULL)
</code></pre>

![alt text](/img/HP/top20words.jpeg "Top 20 words in Harry Potter series")

Unsurprisingly, the most common words in the series are also the names of the principle characters in the series. The words ‘eyes’ and ‘looked’ are the most frequent non-character words, also not too surprising given J.K. Rowling’s writing style.

## Most common words by book
What about the most common words by book? This time we will look at term frequency as a percentage of words within each book, as some books in the series are much longer than others: 

<pre><code class="language-r line-numbers">#find total wordcount in each book
book_wordcounts <- hp_all_books %>% 
  group_by(book) %>% 
  summarize(book_wordcount = max(row_number()))

#graph term frequency as percent of total
hp_tidy_nostop %>%
  group_by(book) %>% 
  count(word, sort = TRUE) %>% 
  top_n(10) %>% 
  arrange(book, desc(n)) %>% 
  left_join(book_wordcounts, by = "book") %>% 
  mutate(term_freq = n/book_wordcount) %>% 
  ggplot(aes(x = fct_rev(fct_infreq(word)), y = term_freq, fill = factor(book))) +
    geom_col() +
    scale_y_continuous(breaks = c(0.01, 0.02), 
                       labels = scales::percent) +
    scale_fill_manual(values = book_color, guide=FALSE) +
    coord_flip() +
    theme_bw() +
    facet_grid(. ~ book, labeller = label_both) +
    #theme(axis.text.x = element_blank()) +
    labs(
      title = "Term Frequency by Book",
      #subtitle = "As percent of total words",
      y="Term Frequency (As % of Total Words)",
      x=NULL,
      fill=NULL)
</code></pre>

![alt text](/img/HP/term freq by book.jpeg "Most common words in each book")

Again, we see Harry, Ron, Hermoine, occupying the top spots in each of the books. We do start to see some variation between the books though towards the bottom of the top 10 lists. 

## Unique words by book
What about the words that makes each book unique? Here we will use a technique called tf-idf, or _term frequency–inverse document frequency_ to identify words which are found most frequently in a single book that are rarely found in the other books. 

<pre><code class="language-r line-numbers">#plot TFIDF unique words by book (facet)
tf_idf <- hp_tidy_nostop %>%
  group_by(book) %>% 
  count(word, sort = TRUE) %>% 
  bind_tf_idf(word, book, n) %>% 
  arrange(book, desc(tf_idf)) %>% 
  top_n(10) %>% 
  ungroup() %>% 
  mutate(bookword = factor(paste0(book, word)) %>% 
           fct_inorder())
tf_idf %>% 
  ggplot(aes(x = fct_rev(bookword), y = tf_idf, fill = factor(book))) +
  geom_col() +
  scale_fill_manual(values = book_color, guide=FALSE) +
  scale_x_discrete(labels = setNames(tf_idf$word, tf_idf$bookword)) +
  coord_flip() + 
  theme_bw() +
  facet_wrap(~book, ncol = 2, scales = "free", labeller = label_both) +
  labs(
    title = "Top 10 Unique Words by Book",
    subtitle = "As Measured by Term Frequency / Inverse Document Frequency",
    y="TF-IDF Weight",
    x=NULL,
    fill=NULL)
</code></pre>

![alt text](/img/HP/tfidf.JPG "TF-IDF")

I like this chart because it really gives a good sense of some of the most important characters and events that were unique to each book.



### Sentiment over time

Another interesting analysis we can do is to look at the sentiment of the words in the book over time to see if we can use those sentiment scores to quantify the ups and downs within a story arc. 

In order to do this, we’ll leverage two common sentiment lexicons (AFINN and Bing), which we will match to the words in the Harry Potter series. The AFINN lexicon assigns each word a sentiment score between _-5_ and _5_, while the Bing lexicon simply assigns each word the value of _Positive_ or _Negative_. Words that are not included in the sentiment lexicons are assumed to be neutral.
<pre><code class="language-r line-numbers">#tidytext sentiment analysis create datasets
tidy_sent_bing <- hp_tidy_nostop %>% 
  inner_join(get_sentiments("bing")) %>% 
  count(book, pagenum, sentiment) %>% 
  spread(sentiment, n, fill = 0) %>%
  mutate(sentiment_bing = positive - negative) %>% 
  select(-c(positive, negative))

tidy_sent_afinn <- hp_tidy_nostop %>% 
  inner_join(get_sentiments("afinn")) %>% 
  group_by(book, pagenum) %>% 
  summarize(sentiment_afinn = sum(score)) 

hp_tidy_sent <- tidy_sent_bing %>% 
  left_join(tidy_sent_afinn) %>% 
  mutate(bing = scale(sentiment_bing)) %>% 
  mutate(afinn = scale(sentiment_afinn)) %>%
  mutate(afinn = ifelse(is.na(afinn), 0, afinn)) %>% 
  select(-starts_with("sentiment")) %>% 
  gather(`bing`, `afinn`, key = "lexicon", value = "sentiment")

#histogram of sentiment by book and lexicon
hp_tidy_sent %>% 
  ggplot(aes(x = pagenum, y = sentiment, fill = factor(book))) +
    geom_col() +
    scale_y_continuous(limits = c(-4, 4)) +
    facet_grid(lexicon ~ book, scales = "free_x", 
               labeller = labeller(.rows = label_both, .cols = label_both),
               switch = "y") +
    scale_fill_manual(values = book_color, guide=FALSE) +
    theme_bw() +
    theme(axis.text.x = element_blank(),
          axis.ticks.x = element_blank(),
          panel.grid = element_blank()) + 
    labs(
      title = "Page Level Sentiment",
      subtitle = "By Book and Sentiment Lexicon",
      y="Normalized Sentiment Score",
      x="Page Number",
      fill=NULL)
</code></pre>

After assigning sentiment scores to each word, I calculated the page-level sentiment by taking the sum of the scores on each page. Graphing the scores at the page level is messy, but looks like this: 

![alt text](/img/HP/page_level_sentiment.JPG "Sentiment by Page")

As that chart is not very illustrative of sentiment changes over time, let’s see if we can further analyze the story arc by fitting a more simple curve to this sentiment data. To do this, I used a discrete cosine transformation described by Matthew Jockers using his [syuzhet package]( https://cran.r-project.org/web/packages/syuzhet/vignettes/syuzhet-vignette.html).
<pre><code class="language-r line-numbers">#loops to make tidy dataset for syuzhet analysis
hp_plot_sentiment <- tibble()
for (bk in seq_along(filenames)){
  for (lx in (c("afinn", "bing"))){
    tmp <- hp_tidy_sent %>% 
      filter(lexicon == lx) %>% 
      filter(book == bk) %$% #explode data
      syuzhet::get_dct_transform(sentiment, scale_range = TRUE) %>% 
      as.numeric() %>% 
      as_tibble() %>% 
      mutate(book = bk) %>% 
      mutate(percentage = row_number()) %>% 
      mutate(lexicon = lx) %>% 
      print()
    
    hp_plot_sentiment <- bind_rows(hp_plot_sentiment, tmp)
  }
}

#create line charts of plot over time using syuzhet methodology
hp_plot_sentiment %>% 
  ggplot(aes(x = percentage, y = value, color = factor(book))) +
  geom_line(size = 2) +
  scale_x_continuous(breaks = c(50, 100),
                     labels = c("50%", "100%")) +
  scale_y_continuous(breaks = c(-1, 0, 1),
                     limits = c(-1, 1)) +
  facet_grid(lexicon ~ book, 
             labeller = labeller(.rows = label_both, .cols = label_both),
             switch = "y") +
  scale_color_manual(values = book_color, guide=FALSE) +
  theme_bw() +
  theme(axis.text.x = element_blank(),
        axis.ticks.x = element_blank()) + 
  labs(
    title = "Discrete Cosine Transformed Plot Trajectory",
    subtitle = "By Book and Sentiment Lexicon",
    y="Emotional Valence",
    x="Narrative Time",
    fill=NULL)
</code></pre>

![alt text](/img/HP/syuzhet.jpeg "Discrete Cosine Transformed Story Arcs")

These simplified sentiment-based plot trajectories are much simpler, and it is interesting to see that both sentiment lexicons lead to similar but not identical plot trajectories. 

However, while interesting and visually appealing, we have no prior baseline for what the plot shape for each of the books should look like. This means that we can’t validate whether or not this curve fitting approach to sentiment is one that leads us to results that make sense. 
