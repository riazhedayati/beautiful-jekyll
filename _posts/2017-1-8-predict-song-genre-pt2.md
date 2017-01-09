---
layout: post
title: Predicting Song Genre using Lyrics (part 2)
subtitle: Cleaning and Exploration in R
---

In case you missed it please see part 1 of this post [here](link).

Now that we have the data, it’s time to clean it and start doing some exploration. Here is a snapshot of our data:

<insert photo of data viewer from R>


##Cleaning the Data
After loading all necessary libraries and reading in the data, we’ll start by splitting the songinfo column into two variables: Artist and Song Title. 

<pre><code class="language-r line-numbers">

#check to see which libraries we really need
library(tm)
library(lsa)
library(fpc)   
library(cluster)
library(rpart)
library(rpart.plot)
library(caTools)
library(wordcloud)
library(RColorBrewer)
library(ggplot2)

#read in lyrics 
lyrics <- read.table("TextMining_Lyrics.txt", stringsAsFactors=FALSE, header=TRUE)

#split songinfo into artist and songname
lyrics$Artist <- sapply(strsplit(lyrics$SongInfo, ' - '), '[', 1)
lyrics$SongTitle <- sapply(strsplit(lyrics$SongInfo, ' - '), '[', 2)
</code></pre>


It looks like there are some songs which are listed in the top 100 of multiple genres. While we could manually decide the more appropriate genre for the duplicated song, for simplicity we are just drop the second instance. 

<pre><code class="language-r line-numbers">str(lyrics)
head(lyrics)
table(lyrics$Genre)

#check for and remove duplicates
lyrics$SongTitle[duplicated(lyrics$SongTitle)]
lyrics <- subset(lyrics, !duplicated(lyrics$SongTitle))
table(lyrics$Genre)
</code></pre>


Although we were aiming to have 100 songs in each of the six genres, it appears that after dropping duplicates and accounting for broken links, our sample size drops to between 87 and 99 per genre.

```r
##   Christian Country Music   Hip Hop/Rap           Pop           R&B          Rock 
##           99            87            91            90            91            91 
```

We’ll also do a bit of feature engineering by creating a variable that contains the wordcount for each song. The minimum number of words for a song in our dataset is 54, the median is 248, and the max is 2936. We’ll also plot the mean wordcount by genre.

<pre><code class="language-r line-numbers">#count number of words in each song
lyrics$WordCount <- sapply(gregexpr("[[:alpha:]]+", lyrics$Lyrics), function(x) sum(x > 0))
summary(lyrics$WordCount)

#create bar graph of avg words by genre
genremean <- data.frame(genre=levels(as.factor(lyrics$Genre)),
                  meanwords=tapply(lyrics$WordCount, lyrics$Genre, mean))

ggplot(genremean, aes(x = factor(genre), y = meanwords)) + geom_bar(stat = "identity") +
  geom_text(aes(y=meanwords, ymax=800, label=round(meanwords,0)), position= position_dodge(width=0.9), vjust=-.5, color="black") +
  scale_y_continuous("Average Wordcount",limits=c(0,800),breaks=seq(0, 800, 200)) + 
  scale_x_discrete("Genre")
</code></pre>

<insert plot>


Finally, we have to clean up the lyrical text itself. We’ll create a variable called corpus using the tm package where we’ll do all of the cleaning. The first four steps to cleaning the corpus are pretty straightforward: 
1.	Make all words lowercase
2.	Remove all formatting
3.	Remove punctuation
4.	Remove any whitespace

<pre><code class="language-r line-numbers"># Create corpus from lyrics and clean it
corpus = Corpus(VectorSource(lyrics$Lyrics))

corpus = tm_map(corpus, tolower) #make words lower-case
corpus = tm_map(corpus, PlainTextDocument) #remove formatting

corpus = tm_map(corpus, removePunctuation) #remove punctuation
corpus <- tm_map(corpus, stripWhitespace) #remove any white space
</code></pre>


We also need to take a few more steps: 
5.	Drop stopwords
6.	Stem words
7.	Remove sparse terms

Stopwords are common words which add no value to the context or meaning of a document(words like ‘the’, ‘and’, ‘that’, ‘which’, etc). 

Stemming words allows us to combine words in the corpus that have the same stem. For instance, the words _love_, _loved_, and _loving_ would be combined into the same root _lov_. 

Finally, in order to make our dataset a more manageable size, we will remove sparse terms, keeping only terms that appear in more than 2% of the songs in our dataset. This brings us from 6762 unique stems down to 785. 

<pre><code class="language-r line-numbers">corpus = tm_map(corpus, removeWords, stopwords("english")) #remove stopwords
dtmcloud = DocumentTermMatrix(corpus)
corpus = tm_map(corpus, stemDocument, language="english") #stemming

# create document term matrix
dtm = DocumentTermMatrix(corpus)
dtm

# Remove sparse terms
dtm = removeSparseTerms(dtm, 0.98)
dtm #785 terms remain, with sparcity of 93%
</code></pre>


Finally, we’ll create some visualizations to better understand the lyrics of all the songs in our data. We’ll create a word cloud using the top 40 most common words, as well as a barchart of the top 20 most common words. Both visualizations are created using the cleaned dataset before stemming, as stemming did not affect the results. 

<pre><code class="language-r line-numbers"># create word cloud
freq <- sort(colSums(as.matrix(dtmcloud)), decreasing=TRUE)
dark2 <- brewer.pal(8, "Dark2")   
wordcloud(names(freq), freq, max.words=40, rot.per=0.2, colors=dark2)  

# plot top 20 words
wf <- data.frame(word=names(freq), freq=freq)   
ggplot(subset(wf, freq>450), aes(word, freq)) + 
  geom_bar(stat="identity") +  
  theme(axis.text.x=element_text(angle=45, hjust=1))  + 
  labs(list(title = "Frequency of Top 20 Words", x = "Word", y ="Frequency"))
</code></pre>

<insert wordcloud and barchart>

Now that we have a cleaned dataset, we can start making some predictions. Please see part 3 of the post [here](link).


