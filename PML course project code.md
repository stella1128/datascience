## Course Project
setwd("/Users/Serena/Courses/Practical Machine Learning (Coursera)/project")

library(caret)
library(eeptools)
library(chron)
library(pROC)
library(ggplot2)

data.train <- read.csv("pml-training.csv", header=T, stringsAsFactors=F)
data.test <- read.csv("pml-testing.csv", header=T, stringsAsFactors=F)

### data cleaning

# recode missing values
sum(is.na(data.train))
data.train[data.train==""] <- NA
sum(is.na(data.train))

# calculate missing rate for each variable
table(apply(data.train, 2, function(x){sum(is.na(x))/length(x)})) #100 variables with ~98% missing rate, which should be removed
d.train <- data.train[ ,apply(data.train, 2, function(x){all(!is.na(x))})]
d.test <- data.test[ ,apply(data.train, 2, function(x){all(!is.na(x))})]

### pre-processing 

## combine training and testing datasets
d <- rbind(cbind(d.train[,-ncol(d.train)], set="training"), cbind(d.test[,-ncol(d.test)], set="testing"))

# recode binary variable - new_window
d$new_window <- ifelse(d$new_window=="yes", 1, 0)

# histogram for each continuous variable
var.remove <- c("X","user_name","cvtd_timestamp","set","classe")
var.cont <- colnames(d)[!is.element(colnames(d), var.remove)]
for(i in var.cont){
	pdf(paste("histograms/", i, ".pdf", sep=""))
	hist(d[,i], xlab=i, main=i)
	dev.off()
}
# possible outliers?
# gyros_dumbbell_x -204
# gyros_dumbbell_y 52
# gyros_dumbbell_z 317
# gyros_forearm_x -22
# gyros_forearm_y 311
# gyros_forearm_z 231
# magnet_dumbbell_y -3600
# accel_dumbbell_x -200
# accel_dumbbell_z -300
# magnet_dumbbell_y -4000

# 1) Predictors 
data <- d[,!is.element(colnames(d), var.remove)]
colnames(data)[nearZeroVar(data)]	#near zero variance
#new_window
colnames(data)[findCorrelation(cor(data),0.9)]	#multicollinearity
#"accel_belt_z"     "roll_belt"        "accel_belt_y"     "accel_belt_x"     "gyros_arm_y"      "gyros_forearm_z" "gyros_dumbbell_x"
findLinearCombos(data)	#linear combinations

# 2) Outcome
table(d.train$classe)
o <- data.frame(classe=d.train$classe, user=d.train$user_name, timestamp=strptime(d.train$cvtd_timestamp, "%d/%m/%Y %H:%M"))
o[,c("date","time")] <- do.call(rbind, strsplit(as.character(o$timestamp), " "))
write.table(o, "outcome.txt", quote=F, sep="\t", row.name=F, col.name=T)

# on sailfish
o <- read.delim("outcome.txt", header=T, stringsAsFactors=F)
library(ggplot2)
pdf("classe_by_user.pdf")
qplot(factor(user), data=o, geom="bar", fill=classe, xlab="User", ylab="Frequency")
dev.off()

## split training and test set
table(d.train$user_name)
table(d.test$user_name)

d.train.qc <- cbind(d[d$set=="training", ], classe=d.train[,ncol(d.train)])
d.test.qc <- d[d$set=="testing", ]

### Modeling
rep <- 5 	#repeats of cross validation
fold <- 5	#folds of cross validation

## penalized multinomial logistic regression
set.seed(1128)
glmFit <- train(d.train.qc[,!is.element(colnames(d.train.qc), var.remove)], d.train.qc$classe, method="multinom", preProc=c("center","scale"), tuneGrid = expand.grid(.decay=seq(0,0.1,by=0.01)), metric="Accuracy", trControl=trainControl(method="repeatedcv", number=fold, repeats=rep, classProbs=T), maxit=1000)
#0.01
save(glmFit, file="fit.RData")

Fit <- glmFit
plot(Fit)
Classes <- predict(Fit, newdata=d.test.qc[,!is.element(colnames(d.test.qc), var.remove)])
varImp(Fit)

### write out result files	
answers = as.character(Classes)
pml_write_files = function(x){   
	n = length(x)   
	for(i in 1:n){     
		filename = paste0("problem_id_",i,".txt")     
		write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)   
	} 
} 
pml_write_files(answers) 

### out-of-sample errors - leave one subject out at each iteration

p <- c()

weightedAvg <- function(v, w){
	sum(v*w)/sum(w)
}

for(user in unique(d.train.qc$user_name)){

	## split the original training dataset into training and validation sets 
	inValid <- (d.train.qc$user_name==user)
	training <- d.train.qc[!inValid,]
	validation <- d.train.qc[inValid,]
	
	## penalized multinomial logistic regression on the training set
	set.seed(1128)
	glmFit <- train(training[,!is.element(colnames(training), var.remove)], training$classe, method="multinom", preProc=c("center","scale"), tuneGrid = expand.grid(.decay=seq(0,0.03,by=0.01)), metric="Accuracy", trControl=trainControl(method="repeatedcv", number=fold, repeats=rep, classProbs=T), maxit=1000)
	#0.01 for adelmo, 0 for the others
	save(glmFit, file=paste(user, "RData", sep="."))

	## prediction on the validation set	
	Fit <- glmFit	
	Classes <- predict(Fit, newdata=validation[,!is.element(colnames(validation), var.remove)])
	Probs <- predict(Fit, newdata= validation[,!is.element(colnames(validation), var.remove)], type="prob")
	CM <- confusionMatrix(data=Classes, validation$classe)
	ROC <- multiclass.roc(validation$classe, as.numeric(apply(cbind(Probs,validation$classe),1,function(x){x[x[6]]})))
	
	# performance metrics
	Performance <- c(CM$overall["Accuracy"], weightedAvg(CM$byClass[ ,"Sensitivity"],table(validation$classe)), weightedAvg(CM$byClass[ ,"Pos Pred Value"],table(validation$classe)), weightedAvg(CM$byClass[ ,"Balanced Accuracy"],table(validation$classe)), ROC$auc)
	names(Performance) <- c("Accuracy","Recall","Precision","BalancedAccuracy","AUC")	
	p <- rbind(p, c(user, Performance))
}

temp <- c()
for(i in 2:5){
	temp <- c(temp, weightedAvg(as.numeric(p[,i]), table(d.train.qc$user_name)[p[,1]]))
}
write.table(rbind(p, c("overall",temp)), "performance.txt", quote=F, sep="\t", row.name=F, col.name=T)


p <- c()
for(user in unique(d.train.qc$user_name)){
	CM <- CMlist[[user]]
	
	Performance <- c(CM$overall["Accuracy"], weightedAvg(CM$byClass[ ,"Sensitivity"],table(validation$classe)), weightedAvg(CM$byClass[ ,"Pos Pred Value"],table(validation$classe)), weightedAvg(CM$byClass[ ,"Balanced Accuracy"],table(validation$classe)))
	names(Performance) <- c("Accuracy","Recall","Precision","BalancedAccuracy")	
	p <- rbind(p, c(user, Performance))
}
