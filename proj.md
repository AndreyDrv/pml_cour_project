---
title: "Prediction of the exercise manner"
author: "Andrey G"
date: "07.2015"
output: html_document
---

##Introduction
In this project will be used the data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways.
The goal of the project is to predict the manner in which the participans did the exercise.

##Executive Summary
The data analysis and all necessary manipulations were made. Feature selection was performed. Three models were tested. Features were optimized for the two of them: Random Forest and Gradient Boosting. Gradient Boosting was selected as more perpective, but Random Forest showed the similar results, but potentially has a better computational performance.

##Data Exploratory Analysis
```{r, echo=FALSE, results='hold'}
library("caret")
library("rpart")
```

```{r, warning=FALSE}
data_src <- read.csv("~/practml_cour/proj/pml-training.csv", na.strings=c("NA","","#DIV/0!"))
data_unittests_src <- read.csv("~/practml_cour/proj/pml-testing.csv", na.strings=c("NA","","#DIV/0!")) #dataset, given as a final unit test 

# cast data to numeric to avoid errors (excepting the last and first 5 fields)
for(i in c(6:ncol(data_src)-1)) {
  data_src[,i] = as.numeric(as.character(data_src[,i]))
  data_unittests_src[,i] = as.numeric(as.character(data_unittests_src[,i]))
}
```
The data contents has `r ncol(data_src)` columns and `r nrow(data_src)` rows. Columns are the values of different sensors data, related timestamps and sensor type, grouped by 'classe' variable. 'Classe' is the manner of the excersise. 
```{r}
str(data_src["classe"])
```
As it is described in the data source. There are 5 levels of this factor. 'A' means that the excersise done exactly according to the specification, the rest factors correspond to different types of the common mistakes.
Our goal is prediction of the 'classe' outcome. The part or all the sensor data should be a predictor.

##Building Model
####Feature Selection and Data Splitting for Cross Validation 
We have so many variables in our datset. 
To improve the feature selection, first, we can try to remove near zero covariates.
```{r}
data_mod <- data_src[-nearZeroVar(data_src)]
data_unittests_src_mod <- data_unittests_src[-nearZeroVar(data_src)]
```
Now we have `r ncol(data_mod)` (`r ncol(data_src)` before) variables as predictor candidates.

It is a good time to make a preprocessing with Principal Component Analysis and shrink our dataset set one more time. First version of this report was using PCA and we had a good results. But throwing away PCA with saving the current set of features optimized a lot Random Forest and Gradient Boosting models below. 

It is important to split our data into training set, test set and validation set.
We need to divide our source dataset in such way because we will use 'Split Sample' method to pick up the best model for our prediction.
```{r}
inTrain <- createDataPartition(y=data_mod$classe,p=0.6, list=FALSE) 
test_and_validation_set <- data_mod[-inTrain,]
inTest <- createDataPartition(y=test_and_validation_set$classe,p=0.5, list=FALSE) 

trainset      <- na.omit(data_mod[inTrain,])
testset       <- na.omit(test_and_validation_set[inTest,])
validationset <- na.omit(test_and_validation_set[-inTest,])
```
Train (`r nrow(trainset)`), Test (`r nrow(testset)`), Validation (`r nrow(validationset)`) datasets are splitted as 60% x 20% x 20% accordingly.
Train will be used to train the model, Test will be used to verify model.
Validation is used to compare the results of the different models.
The unit test dataset will be used as final model prediction.
Unit test dataset has `r nrow(validationset)` test cases.

####Model Selection
The goal is to select the proper model avoiding overfitting on training data and minimize error on test data as much as possible.
One of the best method to pick prediction model is to split the given data into different test sets. All models will be trained on 'Train' dataset, tested on 'Test' dataset and the best model will be selected after testing on the 'Validation' dataset by picking the most good model results. 

Prediction models to be built and tested:

1. Classification Trees - we can try to construct decision trees and produces nonlinear model. As we have so many predictors, the computing complexity is very high.

2. Random Forest - extension of bagging on classification/regression trees. Can give better accuracy.

3. Stochastic Gradient Boosting - is alongside with the random forest. Widely used method.

Regression models and Probabalistic models were thrown out because Random Forest and Boosting methods showed the acceptable performance results at this step. Please refer to the results below. 

#####Classification trees
```{r}
# fit classification tree as a model
modFit_t <- train(classe ~ .,method="rpart",data=trainset) #,preProcess="pca"

# predict outcome for train data set using the classification trees
predict_t_train <- predict(modFit_t,trainset)
cm_t_train <- confusionMatrix(trainset$classe,predict_t_train)

# predict outcome for test data set using the random forest model
predict_t_test <- predict(modFit_t,testset)
cm_t_test <- confusionMatrix(testset$classe,predict_t_test)

rbind("in sample error (%) = " = (100 - cm_t_train$overall['Accuracy']*100),
      "out of sample error (%) = " = (100 - cm_t_test$overall['Accuracy']*100))
```

#####Random forest
```{r}
# apply random forest
modFit_rf <- train(classe~ .,data=trainset,method="rf",prox=TRUE) #,preProcess="pca" 

# predict outcome for train data set using the random forest model
predict_rf_train <- predict(modFit_rf,trainset)
cm_rf_train <- confusionMatrix(trainset$classe,predict_rf_train)

# predict outcome for test data set using the random forest model
predict_rf_test <- predict(modFit_rf,testset)
cm_rf_test <- confusionMatrix(testset$classe,predict_rf_test)

rbind("in sample error (%) = " = (100 - cm_rf_train$overall['Accuracy']*100),
      "out of sample error (%) = " = (100 - cm_rf_test$overall['Accuracy']*100))
```

#####Stochastic Gradient Boosting
```{r, warning=FALSE}
modFit_sgb <- train(classe ~ ., method="gbm",data=trainset,verbose=FALSE)

# predict outcome for train data set using the random forest model
predict_sgb_train <- predict(modFit_sgb,trainset)
cm_sgb_train <- confusionMatrix(trainset$classe,predict_sgb_train)

# predict outcome for test data set using the random forest model
predict_sgb_test <- predict(modFit_sgb,testset)
cm_sgb_test <- confusionMatrix(testset$classe,predict_sgb_test)

rbind("in sample error (%) = " = (100 - cm_sgb_train$overall['Accuracy']*100),
      "out of sample error (%) = " = (100 - cm_sgb_test$overall['Accuracy']*100))
```
As a result we can say, after training all the models the Gradient Boosting model has the best accuracy with the current set of features. Random Forest is not far behind.


#####Selected Models Evaluation Results
Lets summarize the model choice with the validation dataset 
```{r}
predict_t   <- predict(modFit_t,validationset)
predict_rf  <- predict(modFit_rf,validationset)
predict_sgb <- predict(modFit_sgb,validationset)

rbind("Classification Tree Validation Accuracy (%) = " = confusionMatrix(validationset$classe,predict_t)$overall['Accuracy'],
      "Random Forest Validation Accuracy (%) = " = confusionMatrix(validationset$classe,predict_rf)$overall['Accuracy'],
      "Gradient Boosting Validation Accuracy (%) = " = confusionMatrix(validationset$classe,predict_sgb)$overall['Accuracy']
)
```
Result: Gradient Boosting and Random Forest show similar good results. Gradient Boost selected as more appliable. But Random Forest could be less computational efficient.

####Conclusion
The model was successfully built for detecting the manner in which the participans did the exercise. Random Forest and Gradient Boosting showed the similar good results with out of the sample error = `r (100 - cm_rf_test$overall['Accuracy']*100)` and `r (100 - cm_sgb_test$overall['Accuracy']*100)` accordingly.
