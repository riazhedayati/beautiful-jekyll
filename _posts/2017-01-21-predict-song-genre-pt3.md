---
layout: post
title: Predicting Song Genre using Lyrics (part 3)
subtitle: Decision Trees and Random Forests in R
---

Please see part 2 of this post [HERE](https://riazhedayati.github.io/blog/predict-song-genre-pt2/).

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

## Create Training and Test sets
Before we start building models, we first need to split the data. In this case, splitting the data using a simple random sample should be sufficient. We will use 70% of the data to train the models, then we will apply our models to the other 30% of the data.

Using the test set, we can compare the genre predictions to the true genre for each song and calculate an accuracy rate. This will allow us to see which model performs the best on data not used to build the model, and will help us ensure that our selected model will generalize well to new data.

<pre><code class="language-r line-numbers">set.seed(100)
spl = sample.split(lyrics.tree$genre, 0.70)
lyricsTrain = subset(lyrics.tree, spl == TRUE)
lyricsTest = subset(lyrics.tree, spl == FALSE)
</code></pre>


## Build a Decision Tree
As a first pass at predicting genre we will build a simple [decision tree]( https://en.wikipedia.org/wiki/Decision_tree). Decision trees are a great starting point as they are simple and easily interpretable. We will then use the model to make predictions on the test set. 

<pre><code class="language-r line-numbers"># create decision tree model
lyricsCART = rpart(genre~., data=lyricsTrain, method="class")
prp(lyricsCART)
# predict on test set using CART
predCART = predict(lyricsCART, newdata=lyricsTest, type = "class")
table(lyricsTest$genre, predCART)
</code></pre>

The decision tree tries to create rules that will categorize each song into a genre. For any given song, we can examine its lyrics and follow the classification rules until our song is assigned to a genre. This is what our trained decision tree looks like: 
<! Include image of tree >

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

Let’s look at the confusion matrix. The rows of the matrix are the actual genres and the columns are the predicted genres. Our accuracy is equal to the number we predicted correctly divided by the total number of songs in the test set. For the decision tree, the overall accuracy rate is (20+9+4+11+24+3)/164 = 43.3%.

```					                            PREDICTED
		                    Christian Country Pop R&B Rap Rock
		        Christian        20       3   2   5   0    0
		        Country           7       9   1   3   0    6
  ACTUAL	  Pop               4       9   4   6   1    3
		        R&B               5       8   3  11   0    0
		        Rap               0       0   0   3  24    0
		        Rock              2       5   4  13   0    3
```

The accuracy rates for each genre are as follows:

```
Christian: 66.7%
Country: 34.6%
Rap: 88.9%
Pop: 14.8%
R&B: 40.7%
Rock: 11.1%
```

A naïve model where all songs are classified into the same genre (e.g. _Country_) will predict genre correctly approximately 1 in 6 times, so our model has to be right more than 16.67% of the time in order to be worthwhile. It looks like our decision tree outperforms a naïve model overall, but for the _Pop_ and _Rock_ genres it actually underperforms. 


## Random Forests

For a more accurate prediction we will build a [Random Forest]( https://en.wikipedia.org/wiki/Random_forest), which is essentially just an ensemble of many decision trees which are trained using bootstrapped data and a random subset of variables. While Random Forests tend to lead to better predictions, the downside is that they are more complex and less interpretable. 

<pre><code class="language-r line-numbers"># create randomForest model
lyricsRF <- randomForest(as.factor(genre)~., data=lyricsTrain, importance=TRUE)
varImpPlot(lyricsRF)
# predict on test set using RF
predRF <- predict(lyricsRF, lyricsTest, type="response")
table(lyricsTest$genre, predRF)
</code></pre>

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


```					                           PREDICTED
		                    Christian Country Pop R&B Rap Rock
		        Christian        28       0   0   1   0    1
		        Country           1      20   0   3   0    2
  ACTUAL	  Pop               4       6   5   5   1    6
	      	  R&B               2       7   1  16   0    1
		        Rap               0       0   2   1  24    0
		        Rock              2       6   1   5   2   11
```

As expected, the Random Forest outperforms the simple decision tree, with an overall accuracy rate of 63.4%. The genre-specific accuracy rates also improve:

```
Christian: 93.3%
Country: 76.9%
Rap: 88.9%
Pop: 18.5%
R&B: 59.3%
Rock: 40.7%
```

## In Summary

Our initial goal was to use song lyrics to predict genre, and our best performing model predicted the correct genre about 63% of the time. Moreover, performance is about 3.8 times better than a naïve baseline.

While there are surely some [other](https://en.wikipedia.org/wiki/Tf%E2%80%93idf) [manipulations](https://en.wikipedia.org/wiki/Latent_semantic_analysis) and [models](https://en.wikipedia.org/wiki/Gradient_boosting) we could try to make incremental improvements to our accuracy rate, we at least know predicting genre using song lyrics is possible to some extent.
