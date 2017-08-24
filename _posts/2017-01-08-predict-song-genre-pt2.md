---
layout: post
title: Can a song's lyrics predict it's genre? (part 2)
subtitle: Cleaning, Exploration, Predictions in R
---

Please see part 1 of this post [HERE](https://riazhedayati.github.io/blog/predict-song-genre-pt1/).

Now that we have collected the data, it’s time to start doing some cleaning and exploration in R. Here is a snapshot of what our data looks like:

![Alt text](/img/songlyrics/lyricdatasnapshot.JPG "Data Snapshot")




## Cleaning the Data
After loading all necessary libraries and reading in the data, we’ll start by splitting the songinfo column into two variables: _Artist_ and _Song Title_. We'll also recode the names of a couple of genres.

<pre><code class="language-r line-numbers">library(ggplot2)
library(tm)
library(RColorBrewer)
library(wordcloud)

#read in lyrics 
lyrics <- read.csv("textminingAllLyrics.csv", stringsAsFactors=FALSE, header=TRUE)

#split songinfo into artist and songname
lyrics$Artist <- sapply(strsplit(lyrics$SongInfo, ' - '), '[', 1)
lyrics$SongTitle <- sapply(strsplit(lyrics$SongInfo, ' - '), '[', 2)

#rename genres
lyrics$Genre[lyrics$Genre=="Country Music"] <- "Country"
lyrics$Genre[lyrics$Genre=="Hip Hop/Rap"] <- "Rap"
</code></pre>



It looks like there are some songs which are listed in the top 100 of multiple genres. While we could manually decide the more appropriate genre for each duplicated song, for simplicity we are just going to drop the second instance. 

<pre><code class="language-r line-numbers">#check for and remove duplicates
lyrics$SongTitle[duplicated(lyrics$SongTitle)]
lyrics <- subset(lyrics, !duplicated(lyrics$SongTitle))
table(lyrics$Genre)
</code></pre>


Although we initially thought we were going to have 100 songs in each of the six genres, it appears that after dropping duplicates and accounting for broken links, our sample size drops to between 87 and 99 per genre.

```
##    Christian         Country          Pop           R&B           Rap          Rock 
##           99             87            90            91            91            91 
```



## Feature Engineering
We’ll also do a bit of feature engineering by creating a variable that contains the wordcount for each song. The minimum number of words for a song in our dataset is 54, the median is 248, and the max is 2936. We’ll also plot the mean wordcount by genre.

<pre><code class="language-r line-numbers">#count number of words in each song
lyrics$WordCount <- sapply(gregexpr("[[:alpha:]]+", lyrics$Lyrics), function(x) sum(x > 0))
summary(lyrics$WordCount)

#create bar graph of avg words by genre
genremean <- data.frame(genre=levels(as.factor(lyrics$Genre)),
                  meanwords=tapply(lyrics$WordCount, lyrics$Genre, mean))

ggplot(genremean, aes(x = factor(genre), y = meanwords)) + geom_bar(stat = "identity") +
  geom_text(aes(y=meanwords, ymax=800, label=round(meanwords,0)), 
                position= position_dodge(width=0.9), vjust=-.5, color="black") +
  scale_y_continuous("Average Wordcount",limits=c(0,800),breaks=seq(0, 800, 200)) + 
  scale_x_discrete("Genre")
</code></pre>

### Average Wordcount by Genre
![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")


## Creating the Corpus
Next, we have to clean up the lyrical text itself. We’ll create a variable called corpus using the tm package where we’ll do all of the cleaning. The first four steps to cleaning the corpus are pretty straightforward:
<ol><li value="1">Make all words lowercase</li>
  <li>Remove all formatting</li>
  <li>Remove punctuation</li>
  <li>Remove any whitespace</li>
</ol>


<pre><code class="language-r line-numbers"># Create corpus from lyrics and clean it
corpus <- Corpus(VectorSource(lyrics$Lyrics))

corpus <- tm_map(corpus, tolower) #make words lower-case
corpus <- tm_map(corpus, PlainTextDocument) #remove formatting

corpus <- tm_map(corpus, removePunctuation) #remove punctuation
corpus <- tm_map(corpus, stripWhitespace) #remove any white space
</code></pre>


We also need to take a few more steps: 
<ol><li value="5">Drop stopwords</li>
  <li>Stemming</li>
  <li>Remove sparse terms</li>
</ol>

Stopwords are common words which add no value to the context or meaning of a document (words like _the_, _and_, _which_, etc). 

Stemming words allows us to combine words in the corpus that have the same root. For instance, the words _love_, _loves_, _loved_, and _loving_ would be treated as multiple instances of the same word. 

Finally we will remove sparse terms, keeping only terms that appear in more than 2% of the songs in our dataset. This brings us from 6762 unique stems down to 785. 

<pre><code class="language-r line-numbers">corpus <- tm_map(corpus, removeWords, stopwords("english")) #remove stopwords
dtmcloud <- DocumentTermMatrix(corpus)
corpus <- tm_map(corpus, stemDocument, language="english") #stemming

# create document term matrix
dtm <- DocumentTermMatrix(corpus)
dtm #6762 unique terms

# Remove sparse terms
dtm <- removeSparseTerms(dtm, 0.98)
dtm #785 terms remain
</code></pre>



## Visualizations
We’ll create some visualizations to better understand the most common lyrics of the songs in our dataset, including a word cloud and a barchart of the frequency of the 20 most common words. Both visualizations are created using the cleaned dataset before stemming. 

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



### Wordcloud
![alt text](/img/songlyrics/wordcloud.jpeg "Wordcloud - Top 40 Terms")


### Most Frequent Words
![alt text](/img/songlyrics/top20barplot.jpeg "Top 20 Words")



## Final Data Prep
Before we can start using lyrics to make predictions about song genres, there is still a bit more data prep we need to do. 

Again, we will load necessary libraries, and convert our term matrix containing all of the words and their counts into a data frame. Our dataset now has 549 rows (one for each song in our dataset) and 785 variables, each of which is a word and the number of times it appears in that song.

<pre><code class="language-r line-numbers">library(caTools)
library(rpart)
library(rpart.plot)
library(randomForest)

# Create data frame
lyrics.tree = as.data.frame(as.matrix(dtm))
rownames(lyrics.tree) <- lyrics$SongInfo #rename each row as the associated song
</code></pre>

We need to include our target variable _Genre_ and our generated variable _WordCount_ in this data frame. We will also change a few variable names which are causing errors in the randomForest package. 

<pre><code class="language-r line-numbers">#include genre and wordcount variables into data frame 
lyrics.tree$genre = lyrics$Genre
lyrics.tree$wordcount = lyrics$WordCount

# rename variables which are causing errors with randomForest
names(lyrics.tree)[names(lyrics.tree) == 'break'] <- 'break1'
names(lyrics.tree)[names(lyrics.tree) == 'next'] <- 'next1'
names(lyrics.tree)[names(lyrics.tree) == 'repeat'] <- 'repeat1'
</code></pre>

## Create Training and Test Sets
Before we start building models, we first need to split the data. In this case, splitting the data using a simple random sample should be sufficient. We will use 70% of the data to train the models, then we will apply our models to the other 30% of the data.

Using the test set, we can compare the genre predictions to the true genre for each song and calculate an accuracy rate. This will allow us to see which model performs the best on data not used to build the model, and will help us ensure that our selected model will generalize well to new data.

<pre><code class="language-r line-numbers">set.seed(100)
spl = sample.split(lyrics.tree$genre, 0.70)
lyricsTrain = subset(lyrics.tree, spl == TRUE)
lyricsTest = subset(lyrics.tree, spl == FALSE)
</code></pre>


## Building a Decision Tree
As a first pass at predicting genre we will build a simple [decision tree]( https://en.wikipedia.org/wiki/Decision_tree). Decision trees are a great starting point as they are simple and easily interpretable. We will then use the model to make predictions on the test set. 

<pre><code class="language-r line-numbers"># create decision tree model
lyricsCART = rpart(genre~., data=lyricsTrain, method="class")
prp(lyricsCART)
# predict on test set using CART
predCART = predict(lyricsCART, newdata=lyricsTest, type = "class")
table(lyricsTest$genre, predCART)
</code></pre>

The decision tree tries to create rules that will categorize each song into a genre. For any given song, we can examine its lyrics and follow the classification rules until our song is assigned to a genre. This is what our trained decision tree looks like: 

### Decision Tree
![Alt text](/img/songlyrics/CARTtree.jpeg "Decision Tree")

Now we’ll look at the accuracy of our model. To do this, we will compare our predicted genres with the actual genres of songs in the test set using a [confusion matrix]( https://en.wikipedia.org/wiki/Confusion_matrix). In addition to an overall accuracy rate, we can also calculate an accuracy rate for each genre. 

<pre><code class="language-r line-numbers">#calculate overall accuracy with CART
sum(diag(table(lyricsTest$genre, predCART)))/nrow(lyricsTest)

#Compute accuracy for christian
predCART.christian<-table(lyricsTest$genre=="Christian",predCART=="Christian")
predCART.christian[2,2]/sum(predCART.christian[2,])

#Compute accuracy for country
predCART.country<-table(lyricsTest$genre=="Country",predCART=="Country")
predCART.country[2,2]/sum(predCART.country[2,])

#Compute accuracy for rap
predCART.rap<-table(lyricsTest$genre=="Rap",predCART=="Rap")
predCART.rap[2,2]/sum(predCART.rap[2,])

#Compute accuracy for pop
predCART.pop<-table(lyricsTest$genre=="Pop",predCART=="Pop")
predCART.pop[2,2]/sum(predCART.pop[2,])

#Compute accuracy for R&B
predCART.rnb<-table(lyricsTest$genre=="R&B",predCART=="R&B")
predCART.rnb[2,2]/sum(predCART.rnb[2,])

#Compute accuracy for rock
predCART.rock<-table(lyricsTest$genre=="Rock",predCART=="Rock")
predCART.rock[2,2]/sum(predCART.rock[2,])
</code></pre>

Let’s look at the confusion matrix. The rows of the matrix are the actual genres and the columns are the predicted genres. Our accuracy is equal to the number we predicted correctly divided by the total number of songs in the test set. For the decision tree, the overall accuracy rate is (20 + 9 + 4 + 11 + 24 + 3) / 164 = 43.3%.

```				                 
## 						PREDICTED
## 		                    Christian Country Pop R&B Rap Rock
## 		        Christian        20       3   2   5   0    0
## 		        Country           7       9   1   3   0    6
##   ACTUAL		Pop               4       9   4   6   1    3
## 		        R&B               5       8   3  11   0    0
## 		        Rap               0       0   0   3  24    0
## 		        Rock              2       5   4  13   0    3
```

The accuracy rates for each genre are as follows:

```
##                                   Christian: 66.7%
##                                     Country: 34.6%
##                                         Rap: 88.9%
##                                         Pop: 14.8%
##                                         R&B: 40.7%
##                                        Rock: 11.1%
```

A naïve model where all songs are classified into the same genre (e.g. _Country_) will predict genre correctly approximately 1 in 6 times, so our model has to be right more than 16.67% of the time in order to be worthwhile. It looks like our decision tree outperforms a naïve model overall, but for the _Pop_ and _Rock_ genres it actually underperforms. 


## Random Forests

For a more accurate prediction we will build a [Random Forest]( https://en.wikipedia.org/wiki/Random_forest), which is essentially just an ensemble of many decision trees which are trained using bootstrapped data and a random subset of variables. While Random Forests tend to lead to better predictions, the downside is that they are more complex and less interpretable. There is no similar tree output, but we can generate a list of the most important variables in terms of predictive power.

<pre><code class="language-r line-numbers"># create randomForest model
lyricsRF <- randomForest(as.factor(genre)~., data=lyricsTrain, importance=TRUE)
varImpPlot(lyricsRF)
# predict on test set using RF
predRF <- predict(lyricsRF, lyricsTest, type="response")
table(lyricsTest$genre, predRF)
</code></pre>

### Most Predictive Variables
<center><img src="/img/songlyrics/MeanDecreaseGiniRF.JPG" alt="Mean Decrease Gini" style="width: 70%; height: 70%"></center>

Just like we did with the decision tree, we will also compare our predictions to the actual genres using the same test set. 

<pre><code class="language-r line-numbers">#calculate overall accuracy with RF
sum(diag(table(lyricsTest$genre, predRF)))/nrow(lyricsTest)

#Compute accuracy for christian
predRF.christian<-table(lyricsTest$genre=="Christian",predRF=="Christian")
predRF.christian[2,2]/sum(predRF.christian[2,])

#Compute accuracy for country
predRF.country<-table(lyricsTest$genre=="Country",predRF=="Country")
predRF.country[2,2]/sum(predRF.country[2,])

#Compute accuracy for rap
predRF.rap<-table(lyricsTest$genre=="Rap",predRF=="Rap")
predRF.rap[2,2]/sum(predRF.rap[2,])

#Compute accuracy for pop
predRF.pop<-table(lyricsTest$genre=="Pop",predRF=="Pop")
predRF.pop[2,2]/sum(predRF.pop[2,])

#Compute accuracy for R&B
predRF.rnb<-table(lyricsTest$genre=="R&B",predRF=="R&B")
predRF.rnb[2,2]/sum(predRF.rnb[2,])

#Compute accuracy for rock
predRF.rock<-table(lyricsTest$genre=="Rock",predRF=="Rock")
predRF.rock[2,2]/sum(predRF.rock[2,])
</code></pre>

As expected, the Random Forest outperforms the simple decision tree, with an overall accuracy rate of 63.4%. The genre-specific accuracy rates also improve:

```					                           
##  						PREDICTED
##  		                    Christian Country Pop R&B Rap Rock
## 		        Christian        28       0   0   1   0    1
## 		        Country           1      20   0   3   0    2
##   ACTUAL	 	Pop               4       6   5   5   1    6
## 	      	  	R&B               2       7   1  16   0    1
## 		        Rap               0       0   2   1  24    0
## 		        Rock              2       6   1   5   2   11
```

```
##                                   Christian: 93.3%
##                                     Country: 76.9%
##                                         Rap: 88.9%
##                                         Pop: 18.5%
##                                         R&B: 59.3%
##                                        Rock: 40.7%
```

## In Summary

Our initial goal was to use song lyrics to predict genre, and our best performing model predicted the correct genre about 63.4% of the time. While there are surely some [other](https://en.wikipedia.org/wiki/Tf%E2%80%93idf) [manipulations](https://en.wikipedia.org/wiki/Latent_semantic_analysis) and [models](https://en.wikipedia.org/wiki/Gradient_boosting) we could try to make incremental improvements to our accuracy rate, our model offers a great improvement over the naïve model baseline of 16.7%.
