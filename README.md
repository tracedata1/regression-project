# regression-project
---
title: "project"
author: "liang qingyan"
date: "3/20/2022"
output: pdf_document
---


```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```



```{r}
news0 <- read.csv("OnlineNewsPopularity.csv")
head(news0)

```
### data cleaning


```{r}
#  n_non_stop_words
round(sum(news0$n_non_stop_words>0.99)/length(news0$n_non_stop_words),2)# this variable is almost a singular vaule ,we will drop this variable in our model.

# kw_min_min:Worst Keyword (Min. Shares). It is weird to have negative value in this variable, we will drop this variable since there are too many wrong values.   
sum(news0$kw_min_min==-1)  #22980 cases we cannot treat this as 
sum(news0$kw_avg_min==-1) # 694
sum(news0$kw_avg_max==-1) # 0
cor(news0[20:22]) # since kw_max_min and kw_avg_min are highly related, we will choose in our model


# n_tokens_content 
sum(news0$n_tokens_content==0)/length(news0$n_tokens_content) # 0.02979013, 1181 cases
# we will treat 0 as missing value
```
```{r}
# n_unique_tokens is Rate Of Unique Words In The Content, the value range should be (0,1),any value great than 1 should be considered as wrong.
# indt <- which(news$n_unique_tokens>1)
# wr <- news[indt,]
# wr # we can drop this case in our data
library(tidyverse)

news<-news0%>%
  filter( n_tokens_content>0 & n_unique_tokens<=1) %>%
  select(-kw_min_min,-kw_avg_min,-url,-timedelta,-n_non_stop_words ) # drop non-predictive variables, also singlar variable n_non_stop_words
  
1-nrow(news)/nrow(news0) #  0.02981536
nrow(news0)-nrow(news) # 1182 the total unnormal cases we delete
```


## data exploration 

### data discription

```{r}
names(news)
summary(news)
```





```{r}
library(Hmisc)
par(mfrow = c(3, 2))
hist.data.frame(news) # obviously left or right screwed, a lot of transformations would be needed to both y variable and Xs variables
```




## data relationship exploration

### categorical data exploration
```{r}
lda <- news$LDA_00+news$LDA_01+news$LDA_02+news$LDA_03+news$LDA_04
hist(lda)
## the sum of this five variabals =1, In this case we should drop one of these variables,we drop LDA_00 in our model.
```



```{r}

# recoding the dummy variables
news$channel[news$data_channel_is_lifestyle==1] <-"lifestyle"
news$channel[news$data_channel_is_entertainment==1] <-"entertainment"
news$channel[news$data_channel_is_socmed==1] <-"socmed"
news$channel[news$data_channel_is_tech==1] <-"tech"
news$channel[news$data_channel_is_world==1] <-"world"
news$channel[news$data_channel_is_world==0 & news$data_channel_is_lifestyle==0 & news$data_channel_is_entertainment==0 & news$data_channel_is_socmed==0 & news$data_channel_is_tech==0] <-"other"
news$channel <- factor(news$channel)

ggplot(news, aes(channel,log(shares)))+
  geom_boxplot()

```

```{r}

# recoding the dummy variables
news$weekday[news$weekday_is_monday==1] <-"monday"
news$weekday[news$weekday_is_tuesday==1] <-"tuesday"
news$weekday[news$weekday_is_wednesday==1] <-"wednesday"
news$weekday[news$weekday_is_thursday==1] <-"thursday"
news$weekday[news$weekday_is_friday==1] <-"friday"
news$weekday[news$weekday_is_saturday==1] <-"saturday"
news$weekday[news$weekday_is_sunday==1] <-"sunday"
news$weekday <- factor(news$weekday)

ggplot(news, aes(weekday,log(shares)))+
  geom_boxplot()
ggplot(news, aes(as.factor(is_weekend),log(shares)))+
  geom_boxplot()
# comment: There are no differences through Monday to Friday on log(shares), median log(shares) for weekends is abviously higher than not weekends. We can drop redundant variables and keep is_weekend instead.

```
### drop redundant variables
```{r}
news1 <- news %>%
  select(-weekday_is_monday, -weekday_is_tuesday, -weekday_is_wednesday, -weekday_is_thursday, -weekday_is_friday, -weekday_is_saturday,-weekday_is_sunday,-data_channel_is_entertainment,-
    data_channel_is_bus,-data_channel_is_socmed ,-data_channel_is_tech,-
    data_channel_is_world,-LDA_00,-weekday)
length(news1) # we have 44 variables in our final data set 
names(news1)
```


```{r}
hist(news$kw_max_max)
ggplot(news, aes(kw_max_max,log(shares)))+
  geom_point(cex=0.1)+
  geom_smooth()
```



```{r}
set.seed(123)
n<-nrow(news1)

index_train <- sample(1:n, round(0.7*n))
newstrain <- news1[index_train,]
newstest <- news1[-index_train,]
```


```{r}
lmnew_full <- lm(shares~.,data=newstrain)
summary(lmnew_full)


plot(lmnew_full)
```


## refit the model
### using log- transformation

```{r}
lmnew_full_log <- lm(log(shares)~.,data=newstrain)
summary(lmnew_full_log )
plot(lmnew_full_log)
```


```{r}

lmnew_full_logs <- step(lmnew_full_log,trace=F)
summary(lmnew_full_logs)
plot(lmnew_full_logs)
```



### using boxcox

```{r}
library(MASS)
library(car)
boxcox(lmnew_full)
summary(powerTransform(lmnew_full))

```


```{r}

lmnew_full_t <- lm((((shares ^ -0.22) - 1) / (-0.22))~.,data=newstrain)
summary(lmnew_full_t )

```


```{r}

plot(lmnew_full_t)

```



## improve the model 


```{r}

lmnew_full_ts <- step(lmnew_full_t,trace=F)
summary(lmnew_full_ts)

```

```{r}

plotpoint <- function(z,x,y) {
  ggplot(z,aes(x,y))+
  geom_point(cex=0.2)+
  geom_smooth(se=F)
}

```

```{r}

plotpoint(newstrain,newstrain$n_tokens_title,((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("n_tokens_title")

```

```{r}

plotpoint(newstrain,newstrain$n_non_stop_unique_tokens,((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("n_non_stop_unique_tokens")

```

```{r}

plotpoint(newstrain,sqrt(newstrain$n_tokens_content),((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("n_tokens_content")

```

```{r}

plotpoint(newstrain,sqrt(newstrain$num_hrefs),((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("num_hrefs")

plotpoint(newstrain,newstrain$num_hrefs,((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("num_hrefs")
plotpoint(newstrain,log(newstrain$num_hrefs),((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("num_hrefs")

```

```{r}

plotpoint(newstrain,newstrain$num_self_hrefs,((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("num_self_hrefs")

plotpoint(newstrain,sqrt(newstrain$num_self_hrefs),lmnew_full_ts$fitted.values) + xlab("num_self_hrefs")

```

```{r}

plotpoint(newstrain,sqrt(newstrain$num_videos),((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("num_videos")

plotpoint(newstrain,newstrain$num_videos,log(newstrain$shares)) + xlab("num_videos")

```


```{r}


plotpoint(newstrain,sqrt(newstrain$self_reference_avg_sharess),((newstrain$shares^-0.22) - 1)/(-0.22)) + xlab("self_reference_avg_sharess")

plotpoint(newstrain,newstrain$self_reference_avg_sharess,log(newstrain$shares)) + xlab("self_reference_avg_sharess")

```

```{r}

lmnew_full_ts1 <- lm(formula = (((shares^-0.22) - 1)/(-0.22)) ~ n_tokens_title + 
    poly(sqrt(n_tokens_content),2)  + n_non_stop_unique_tokens + 
    sqrt(num_hrefs) + poly(sqrt(num_self_hrefs),2) + poly(sqrt(num_imgs),2) + poly(sqrt(num_videos),3) + average_token_length + 
    num_keywords +channel +sqrt(kw_max_min)  +  + 
    kw_min_max + kw_avg_max  + kw_max_avg + kw_avg_avg + 
   sqrt(self_reference_min_shares) + poly(sqrt(self_reference_avg_sharess),3) + 
    is_weekend + LDA_01 + LDA_02 + LDA_03 + LDA_04 + global_subjectivity + 
    global_sentiment_polarity + global_rate_positive_words + 
    rate_positive_words + min_positive_polarity + avg_negative_polarity + 
    title_subjectivity + title_sentiment_polarity + abs_title_subjectivity,
    data = newstrain)

summary(lmnew_full_ts1)
plot(lmnew_full_ts1)

```

```{r}

## check colinearity vif 

vif(lmnew_full_ts1)

```



```{r}

### check outliers
p<-42
n<-nrow(newstrain)
plot(hatvalues(lmnew_full_ts1), rstandard(lmnew_full_ts1),
xlab='Leverage', ylab='Standardized Residuals')
abline(v=2*(p+1)/n)

ind <- which(hatvalues(lmnew_full_ts1)>0.06)
newstrain[ind,]

```









```{r}

lmnew_full_ts4 <- step(lmnew_full_ts1,trace=F)
summary(lmnew_full_ts4)
 plot(lmnew_full_ts4)
```

```{r}
# variable importance 
library(vip)
 vip(lmnew_full_ts4, num_features = 30, geom = "point", include_type = TRUE)
```



```{r}

# calculate r_squre by hand
y<-((newstrain$shares^-0.22) - 1)/(-0.22)
SST <-sum((y-mean(y))^2)
Res<- y-lmnew_full_ts4$fitted.values
rss <-sum(Res^2)
Rsquerd_4 <- 1-(rss/SST)
Rsquerd_4


```



## Evaluate model proformance



```{r}

# function to compute RMSE
RMSE <- function(y, y_hat) {
  sqrt(mean((y - y_hat)^2))
}
```



```{r}

lm_pred4 <- predict(lmnew_full_ts4,newstest)
y_test <-((newstest$shares^-0.22) - 1)/(-0.22)
rmse4 <- RMSE(y_test, lm_pred4)
rmse4

```

```{r}


# calculate r_squre by hand 
y<-((newstest$shares^-0.22) - 1)/(-0.22)
SST <-sum((y-mean(y))^2)
Res4<- y-lm_pred4
SSR4 <-sum(Res4^2)
Rsquerd4 <- 1-(SSR4/SST)
Rsquerd4
```



```{r}

lm_pred1 <- predict(lmnew_full_ts1,newstest)
rmse1 <- RMSE(y_test, lm_pred1)
rmse1

y<-((newstest$shares^-0.22) - 1)/(-0.22)
SST <-sum((y-mean(y))^2)
Res1<- y-lm_pred1
SSR1 <-sum(Res1^2)
Rsquerd1 <- 1-(SSR1/SST)
Rsquerd1
```


```{r}

## check the log_transformation model preofence

y_log <-log(newstest$shares)
pred_1og <- predict(lmnew_full_log,newstest)
SST_log <-sum((y_log-mean(y_log))^2)
Res_log<- y_log-pred_1og
SSR_log <-sum(Res_log^2)
Rsquerd_log <- 1-(SSR_log/SST_log)
Rsquerd_log

```








