---
layout: post
title: What’s in your shopping cart tomorrow?
subtitle: Next basket prediction using Personalized Markov Chains
---

[Instacart](https://www.instacart.com/) is a grocery delivery service, who open sourced an (interesting dataset)[https://www.instacart.com/datasets/grocery-shopping-2017] of about 3 million orders, including nearly 50,000 unique products ordered by over 200,000 users. This dataset has many potential applications, but in this post I am going to focus on next basket prediction. Essentially, if we can accurately use a person’s order history to predict what products will be in their next order, then surface those products to the user’s homepage or another convenient location, we are likely to increase conversion rates.

After doing some research on the topic, I found [several]() [white]() [papers]() which detail various methods of determining the most likely products in a subsequent order. However, when I started looking for R packages to help answer questions like these, there was really nothing available. Given this, I decided to write my own implementation of next basket recommendation.

I based my implementation on the idea of personalized markov chains proposed in [this]() paper by Stephen Rendle. Not only did it seem like the seminal paper on the subject, but it was also one of the most explicit in terms of discussing the methodology in a step by step manner. 

## A simple example
Let’s take for example one person who has made four orders, and has purchased three different products: apples, bananas, and carrots. Here is a table of their orders. Our goal is to predict which products will be in order 5.

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

Rendle’s method essentially takes each order in sequence, looking at the products purchased in an order and their relationship to the products purchased in the previous order. By doing this across all orders for an individual user, we can create a transition matrix calculating the likelihood of purchase of each product, given the products in the previous basket. Our blank transition matrix looks like this:

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

Using our example above, cell [1,1] in our transition matrix is the probability that apples in a basket implies apples in the next basket. Cell [2,1] is the probability that bananas in a basket implies apples in the next basket. Finally, cell [1,3] is the probability that apples in a basket implies carrots in the next basket, and so forth. 

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

### Calculating a transition matrix
Mathematically, the formula to calculate each cell looks like this:

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")


In English, it looks more like this

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")


To calculate these values, let’s look at our example again. Cell [2,1], or the probability (Bananas ==> Apples), is equal to 1/3. Bananas in a previous order imply apples in a subsequent order 1 time (green arrow), while bananas appear in 3 total orders excluding the last order (blue circles).

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")


As another example, cell [1,3], or the probability (Apples ==> Carrots), is equal to 2/2. Apples in a previous order imply carrots in a subsequent order 2 times, while apples appear in 2 total orders (excluding the last order). Another way to think about this is that every time an apple appears, carrots appear in a subsequent order.


![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

Given that we now know how to calculate the transition matrix, we can calculate the rest of the cells:
![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

### Making Predictions
Once we have our transition matrix, we can apply those probabilities to our most recent basket, in an attempt to predict the likelihood of each product in our next order. The table below shows our most recent order from our example, along with the next order which we are trying to predict. 

![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

Applying the probabilities from the transition matrix, to order 4, we get the following probabilities for each product in order 5: 
![alt text](/img/songlyrics/wordcountbygenre.jpeg "Average Wordcount by Genre")

## Testing the results
Now that we’ve gone through a simple example to understand how to use Rendle’s concept of personalized markov chains to predict products in a next basket, let’s apply it to the real data that instacart has released. While the instacart dataset includes order history for over 200,000 customers, we will just take a sample of 3200 customers to test the effectiveness of our personalized markov chain model. We will calculate the effectiveness of our model using an [F1 score](), which can range from 0 to 1. As for interpretability, we can think of an F1 score of 0.5 as the ability to predict the items in the next order with 50% accuracy. 

### Establishing a baseline
Before we test our model, we should establish a simple baseline that we are looking to beat. To create a simple baseline for this problem, we can use the most recent order as the predicted next order. For instance, if someone purchased carrots, kiwis, and milk in their most recent order, our simple model would predict that they will purchase carrots, kiwis, and milk in their next order too. 
Running this ‘most recent order’ model on the sample of 3200 customers, we calculate an average F1 score of 0.2668645.

### Comparing the Personalized Markov Chain model
The outputs of our Personalized Markov Chains are the probabilities that a user will purchase each product in their next order, given their last order. However, we are trying to determine what products will be in someone’s cart, not just the probability that they will be in the cart. Therefore we also need to determine how many products we think will be purchased in a person’s next order. While there are more sophisticated techniques we could use to determine this, we will simply take the average number of products purchased across a user’s order history as our guess.
Using our Personalized markov chain model on our sample of 3200 customers, we have an average F1 score of 0.2672749. 
While the Personalized Markov Chain model does perform slightly better than the simple most recent order model, the effect is miniscule. The Personalized Markov Chain model is only 0.15% more accurate than the baseline model.
 
## Final thoughts
While it was worth a try, it seems that Personalized Markov chains only provides a slight boost in accuracy over a naïve model given this dataset. Theoretically the model will perform better in situations where there are almost cyclical/seasonal patterns of repeated buying. Thinking about how I shop for groceries, what I am likely to purchase on my next shopping trip is not necessarily related to what I purchased during my most recent trip to the store. If I made Italian food last night and am making Mexican food tonight, that doesn’t mean that every time I make Italian food I will make Mexican food the following night. 

All data and R code to reproduce this analysis can be found [here]().