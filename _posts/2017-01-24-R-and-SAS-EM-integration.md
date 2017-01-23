---
layout: post
title: Predicting Adult Income using Demographic Data
subtitle: Open Source Integration in SAS Enterprise Miner
---

Today we’re going to look at how to compare R and SAS models within SAS Enterprise Miner.

To start, we’ll use the [adult]( https://archive.ics.uci.edu/ml/machine-learning-databases/adult/) dataset from the UCI machine learning database. This dataset includes information on 32,561 adults taken from a US census database in 1994. There are a number of demographic variables about each person, and a binary target variable indicating if their income was greater than $50K. 
I really like this dataset, because it’s pretty easy to understand even for people who are new to predictive analytics. This is what our dataset looks like: 
<center><img src="/img/adultincome/adultincomesnapshot.JPG" alt="x" style="width: 70%; height: 70%"></center>

Our target is the variable _income_, where 1 means that individual had an income of >$50K, and 0 means they did not. In this dataset, only about 25% of people had an income >$50K. The goal of our analysis is to build a model using demographic characteristics to predict if someone makes more than $50K. 

First, we’ll clean the data. In order to save some time on the data cleaning portion of this exercise, I used [some code from the SDSU Statistical Consulting Group’s website]( http://scg.sdsu.edu/dataset-adult_r/). They have a great post detailing the steps they took in detail, but essentially they simply drop a couple variables, bin like responses within other variables, drop observations with missing values, and change variable classes to factors.

## Building Models in R
Now that the data is clean, we’ll split it into training (70%) and validation (30%) datasets before we start building models. We will build a decision tree, a logistic regression, and a random forest in R using our training data. Once we build a model, we will use the validation data to make sure that our model was not overfitting our training set and we will calculate a misclassification rate. The model with the lowest misclassification rate is the one we will consider to be the best performing model.
We will begin with a simple decision tree:
<pre><code class="language-r line-numbers"> #build decision tree 
tree_model <- rpart(income ~ type_employer + education + marital + occupation + 
                     relationship + race + sex + capital_gain + capital_loss + 
                     country + age + hr_per_week, data= train, method= "class")
summary(tree_model) #detailed summary of splits
prp(tree_model) #plot the tree

#make predictions on validation dataset using tree model
predicted_income = predict(tree_model, newdata = validation, type="class")
table(validation$income, predicted_income)
accuracydt <- round(sum(predicted_income==validation$income)/nrow(validation) * 100,2)
print(paste("The misclassification rate on validation data with a simple tree is ", 100-accuracydt, "%", sep="")) 
</code></pre>

The decision tree has a validation misclassification rate of 16.1%. 
This means that using our decision tree model, when we made predictions about the incomes of individuals in the validation dataset, and then compared our predictions to the truth (if that person actually made more than $50K), our decision tree model was wrong about 16% of the time.

Now let’s try a logistic regression and a random forest:

<pre><code class="language-r line-numbers"> #build logistic regression 
regression_model <- glm(income ~ type_employer + education + marital + occupation + 
                    relationship + race + sex + capital_gain + capital_loss + 
                    country + age + hr_per_week, data= train, family= binomial())
summary(regression_model)

#make predictions on validation dataset using regression model
predicted_income = round(predict(regression_model, newdata = validation, type="response"),0)
table(validation$income, predicted_income)
accuracyreg <- round(sum(predicted_income==validation$income)/nrow(validation) * 100,2)
print(paste("The misclassification rate on validation data with a simple tree is ", 100-accuracyreg, "%", sep=""))
</code></pre>

<pre><code class="language-r line-numbers"> #build random forest 
trainforest<- train
validforest<- validation  
trainforest$income <- as.factor(trainforest$income)
validforest$income <- as.factor(validforest$income)

set.seed(12345)
forest_model <- randomForest(income ~ type_employer + education + marital + occupation + 
                    relationship + race + sex + capital_gain + capital_loss + 
                    country + age + hr_per_week, data= trainforest)

#make predictions on validation dataset using forest model
print(forest_model) 
importance(forest_model) # importance of each predictor
predicted_income = predict(forest_model, newdata = validforest, type="class")
table(validforest$income, predicted_income)
accuracyrf <- round(sum(predicted_income==validforest$income)/nrow(validforest) * 100,2)
print(paste("The misclassification rate on validation data with a random forest is ", 100-accuracyrf, "%", sep=""))
</code></pre>


The logistic regression had a misclassification rate of 15.6%, and the random forest had a misclassification rate of 15.4%. When comparing our misclassification rates, we can see that the random forest was the best performing model on this data.

## Building Models in SAS Enterprise Miner
Now that we have calculated these three models in R, let’s use SAS Enterprise Miner to build the same models and compare the results. 
The first step in SAS Enterprise Miner is to build a workflow. We will drag and drop the same cleaned data into our workspace, then connect it to the Data Partition node and perform a 70/30 split. We’ll connect it to the Data Mining Database (DMDB) node to check summary statistics on our variables and make sure everything looks as expected. The summary statistics look as expected, with 2 numeric variables and 10 categorical variables, including our categorical target variable. 

###Summary Statistics 
<center><img src="/img/adultincome/DMDBsummarystatistics.JPG" alt="x" style="width: 70%; height: 70%"></center>

Next, we’ll start modeling in SAS. To do this, we’ll drag and drop the Regression, Decision Tree, and Forest nodes onto the workspace and connect them to the workflow. We can connect them to the Model Comparison node to see which performs the best. This is what the workflow currently looks like:

### SAS Model Workflow
<center><img src="/img/adultincome/R_EM_workflow0.JPG" alt="x" style="width: 70%; height: 70%"></center>

## Adding R models to SAS Enterprise Miner
Before we investigate the results of the Model Comparison node, let’s add in the R models we just built as well. To do so, we’ll drag three instances of the Open Source Integration node into our workflow, one for each of the three R models. 
Unfortunately we can’t just copy our R code directly into in the Open Source node; we have to change some of the hard-coded variables in R into macro variables that Enterprise Miner can understand. This is not that hard though. For the R decision tree, our model looked like this:
<pre><code class="language-r line-numbers">tree_model <- rpart(income ~ type_employer + education + marital + occupation + 
                     relationship + race + sex + capital_gain + capital_loss + 
                     country + age + hr_per_week, data= train, method= "class")
</code></pre>
The enterprise miner code node requires macro variables for the model name, variable names, and dataset name. We will change the model name from _tree_model_ to the SAS macro variable _&EMR_MODEL_. Likewise, we will change our target _income_ to _&EMR_CLASS_TARGET_, and our dataset name from _train_ to _&EMR_IMPORT_DATA_. Finally, the dependent variables are grouped into categorical or numeric variables: _&EMR_NUM_INPUT_ in SAS encompasses the numeric variables _age_ and _hr_per_week_ in R, while _&EMR_CLASS_INPUT_ encompasses the rest of the dependent variables which are all categorical. 
For the R decision tree in SAS Enterprise Miner, our model looks like this:
<pre><code class="language-sas line-numbers">&EMR_MODEL <- rpart(&EMR_CLASS_TARGET ~ &EMR_CLASS_INPUT + 
&EMR_NUM_INPUT, data= &EMR_IMPORT_DATA, method= "class")
</code></pre>
<center><img src="/img/adultincome/RtreeinEMcode.JPG" alt="x" style="width: 70%; height: 70%"></center> 

We will rewrite the code from each of our three R models using SAS macro variables the same way. While the rpart and glm packages in R are written in a way that utilizes the predictive modeling markup language ([PMML]( https://en.wikipedia.org/wiki/Predictive_Model_Markup_Language)), the randomForest package is not. SAS Enterprise Miner can interpret PMML output directly, meaning that the results of the R Decision Tree and Regression can be interpreted directly by the Model Comparison node. For the Random Forest which does not support PMML, we will have to add a Model Import node to translate the results to something that the Model Comparison node can understand. Luckily, all that means is we have to assign the output macro variables of the R Random Forest to a predicted variable label manually: 

### Mapping Editor
<center><img src="/img/adultincome/EMmappingeditor.JPG" alt="x" style="width: 70%; height: 70%"></center>

Now that we have each of our three SAS models and our three R models, we will connect all models to the Model Comparison node and check the results. This is what our final workflow looks like:

### Final Workflow
<center><img src="/img/adultincome/R_EM_Workflow.JPG" alt="x" style="width: 70%; height: 70%"></center> 


## Model Comparison
Checking the results of the Model Comparison node, we have several charts and panels. The two most interesting are the ROC charts and the fit statistics table. 

### ROC Curves
<center><img src="/img/adultincome/R_EM_ROC.JPG" alt="x" style="width: 70%; height: 70%"></center> 

Looking at the ROC curves for the training data, it appears that the Random Forest in R has a measurably higher [AUC]() than the other models. Interestingly though, this does not hold true for the validation data, implying that the R random forest implementation is overfitting the training data.
The fit statistics table provides a number of ways by which to measure model performance. As we have selected validation misclassification rate as the decision metric of choice, it appears that the SAS Forest is the best performing model, with a misclassification rate of under 15.1%. 

### Fit Statistics
<center><img src="/img/adultincome/R_EM_FitStatistics.JPG" alt="x" style="width: 70%; height: 70%"></center> 

## Summary
Today we built three models in R and three models in SAS Enterprise Miner, with the goal of using demographic information to predict whether an individual’s income is over $50K. We were able to compare our six models within SAS Enterprise Miner by utilizing the Open Source Integration Node. While the misclassification rate of all the models was in the same ballpark, the best performing model on this dataset was the SAS Forest, with a misclassification rate of 15.07%. 
