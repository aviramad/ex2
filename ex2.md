---
title: תרגיל 2 - סיווג נוסעי הטיטאניק
author: אבירם אדירי - 302991468 , ליהיא ורצ'יק - 308089333
date: "23/4/2017"
---

## רקע ##
בתרגיל זה התבקשנו לסווג את נוסעי הטיטאניק ע"פ נתונים חלקיים.
לשם כך ביצענו מספר נסיונות סיווג, להלן העיקריים שביניהם:
- שימוש ביער רנדומלי תוך ביצוע עיבוד מקדים מינימאלי ושימוש במעט פרמטרים
- שימוש ביער רנדומלי תוך ביצוע עיבוד מקדים מעמיק יותר וכן שימוש במספר רב של פרמטרים
- שימוש במודל לינארי (glm)

## תיאור הנתונים ##

|Feature |  Description|
|-------------|:------------------------------------------------|
|survival |  Survival (0 = No; 1 = Yes)|
|pclass |  Passenger Class (1 = 1st; 2 = 2nd; 3 = 3rd)|
|name  |  Name|
|sex |   Sex|
|age |  Age|
|sibsp  |   Number of Siblings/Spouses Aboard|
|parch  |   Number of Parents/Children Aboard|
|ticket |   Ticket Number|
|fare  |   Passenger Fare|
|cabin  |   Cabin|
|embarked |   Port of Embarkation (C = Cherbourg; Q = Queenstown; S = Southampton)|

## 1 - Random Tree ##
### קריאת נתונים ועיבוד מקדים ###
ראשית ביצענו קריאה של הנתונים, וכן ביצענו המרה לפאקטור ל2 מן השדות:
- survived
- Pclass
כמו כן, ביצענו גם טעינה של כל החבילות הרלוונטיות

```{r}
setwd("~/r_projects/ass2")
library(ggplot2)
library(knitr)
library(caret)
library('randomForest')

train <- read.csv("train.csv",na.strings = "")
test <- read.csv("test.csv",na.strings = "")

set.seed(123)
train$Survived<- as.factor(train$Survived)
train$Pclass<- as.factor(train$Pclass)
test$Pclass <- as.factor(test$Pclass)
```
הערה: קיימים מספר שדות בעלי פרמטרים ריקים, כמו Age וEmbarked, אך איננו משתמשים בשדות אלו בהרצה זו ועל כן אין צורך לטפל בהם כעת

כעת נחלק את הנתונים שלנו לסט בדיקה וסט ולידציה:

```{r}
train <- train[,-c(1,4,9)]
indices <- sample(1:nrow(train),nrow(train)*0.75)
train<- train[indices,]
validation<- train[-indices,]
```

### אימון האלגוריתם על סט הבדיקה ואימות באמצעות סט הוולידציה ###
נבצע אימון לאלגוריתם על סמך המידע שברשותנו.
נשים לב שהשדות שנבדוק הם:
- Pclass
- Sex

```{r}
tc = trainControl(method="cv", number=5)
rf <- train(Survived ~ Pclass+Sex, data=train, method="rf", ntree=2000, trControl=tc, tuneGrid=expand.grid(.mtry=seq(1,7,1)))
```

נבצע כעת אימות של האלגוריתם שלנו על באמצעות סט הולידציה:

```{r}
pred <- predict(rf,validation)
table(pred,validation$Survived)
mean(pred==validation$Survived)
```

### סיווג סט הבחינה וכתיבה לקובץ ###
```{r}
rf.pred<- predict(rf,  newdata=test, type="raw")
result <- cbind(PassengerId = test$PassengerId, Survived=as.character(rf.pred))
write.csv(result,file="randomForest_pred1.csv",row.names = F)
```

### תוצאות הסיווג ###
![image](/images/try1.png)

## 2 - Random Tree with pre-preccessing ##
### קריאת נתונים ועיבוד מקדים ###
קריאת הנתונים בשלב זה זהה לקריאה בהרצאה הראשונה.
לאחר קריאת הנתונים, נבצע מילוי של השדות הריקיים הרלוונטים, והם 
- Embarked
- Age
עבור השדה Age, נחשב את הגיל הממוצע בהתפלגות לפינשים וגברים, עבור כל אחד מסט הנתונים שלנו.
לאחר מכן, נבצע השמה של הערך הממוצע בכל השדות הריקים:

```{r}
train$Age[which(is.na(train$Age) | train$Age=="")] <- -1
female_train = train[train$Sex=="female" & train$Age > 0,] 
qplot(female_train$Age, main="female Age", xlab = "Age", bins = 30)
avg_female_age_train = mean(female_train$Age)
train$Age[which(train$Age==-1)] <- avg_female_age_train

test$Age[which(is.na(test$Age) | test$Age=="")] <- -1
female_test= test[test$Sex=="female" & test$Age > 0,] 
qplot(female_test$Age, main="female Age", xlab = "Age", bins = 30)
avg_female_age_test = mean(female_test$Age)
test$Age[which(test$Age==-1)] <- avg_female_age_test

train$Age[which(is.na(train$Age) | train$Age=="")] <- -1
male_train = train[train$Sex=="male" & train$Age > 0,] 
qplot(male_train$Age, main="male Age", xlab = "Age", bins = 30)
avg_male_age_train = mean(male_train$Age)
train$Age[which(train$Age==-1)] <- avg_male_age_train

test$Age[which(is.na(test$Age) | test$Age=="")] <- -1
male_test= test[test$Sex=="male" & test$Age > 0,] 
qplot(male_test$Age, main="male Age", xlab = "Age", bins = 30)
avg_male_age_test = mean(male_test$Age)
test$Age[which(test$Age==-1)] <- avg_male_age_test
```

עבור השדה Embarked, מכיוון שאין מדובר בשדה מספרי וכן למעט מאוד רשומות חסר השדה הזה, נבצע השמה שרירותית של ערך כלשהו:

```{r}
train$Embarked[which(is.na(train$Embarked) | train$Embarked=="")] <- 'S'
test$Embarked[which(is.na(test$Embarked) | test$Embarked=="")] <- 'S'
```

### אימון האלגוריתם על סט האימון ###
נבצע אימון לאלגוריתם על סמך המידע שברשותנו.
השדות שנבדוק בשלב זה יהיו:
-Pclass
-Age
-SibSp
-Parch
-Sex
-Embarked

```{r}
tc = trainControl(method="cv", number=5)
rf <- train(Survived ~ Pclass+Age+SibSp+Parch+Sex+Embarked, data=train, method="rf", ntree=2000, trControl=tc, tuneGrid=expand.grid(.mtry=seq(1,7,1)))
```

נבצע כעת אימות של האלגוריתם שלנו על באמצעות סט הולידציה:

```{r}
pred <- predict(rf,validation)
table(pred,validation$Survived)
mean(pred==validation$Survived)
```

### סיווג סט הבחינה וכתיבה לקובץ ###
```{r}
rf.pred<- predict(rf,  newdata=test, type="raw")
result <- cbind(PassengerId = test$PassengerId, Survived=as.character(rf.pred))
write.csv(result,file="randomForest_pred1.csv",row.names = F)
```

### תוצאות הסיווג ###
![image](/images/try2.png)

## 3 - GLM with data-preccessing ##
### קריאת נתונים ועיבוד מקדים ###
קריאת הנתונים ועיבודם זהה להרצאה השניה שביצענו

### אימון האלגוריתם על סט האימון ###
נבצע אימון לאלגוריתם על סמך המידע שברשותנו.
השדות שנבדוק בשלב זה יהיו:
-Pclass
-Age
-SibSp
-Parch
-Sex
-Embarked

```{r}
glm <- glm(Survived ~ Pclass+Age+SibSp+Embarked+Sex+Parch, data=train, family=binomial)
```

נבצע כעת אימות של האלגוריתם שלנו על באמצעות סט הולידציה:

```{r}
pred <- predict(glm,validation)
table(pred,validation$Survived)
mean(pred==validation$Survived)
```

### סיווג סט הבחינה וכתיבה לקובץ ###
```{r}
glm.pred<-predict(glm,  newdata=test, type="response")
test$Survived <- as.numeric(as.numeric(glm.pred)>0.55)
res <- cbind(PassengerId = test$PassengerId,Survived=as.numeric(as.numeric(glm.pred)>0.55))
write.csv(res,"glm.csv", row.names=F)
```

### תוצאות הסיווג ###
![image](/images/try3.png)

## סיכום ומסקנות ##
מכל הבדיקות שביצענו המסקנה העיקרית שלנו היתה שפרמטר המין הינו המשפיע ביותר, וכן ככל שלקחנו בחשבון יותר שדות כך התוצאות השתפרו.
מכאן אנו גם מבינים כי שלב עיבוד המידע הינו חשוב מאוד, שכן במידע וחסרים פרטים רבים האופן בו נשלים אותם הינו משמעותי וחשוב.
על כן יש צורך להפעיל מחשבה רבה באותן בו אנו משלימים את הפרטים החסרים.
בבדיקות שביצענו התוצאה הטובה ביותר היתה באמצעות שימוש ביער רנדומלי, כאשר ביצענו עיבוד מקדים טוב וכן כאשר התייחסנו למספר רב יותר של פרמטרים.
ייתכן ואם שינינו באופן שונה את הפרמטרים היינו מגיעים לתוצאות טובות יותר עבור אלגוריתמים אחרים.