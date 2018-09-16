# Predicting the Ideological Direction of Supreme Court Decisons: Ensembles of Justice-Based Models vs. Simple Case-Based Models

## Summary of Findings

By applying a range of supervised machine-learning techniques in scikit-learn - SVM, decision trees and their derivatives (random forests, xgboost, adaboost), KNN clustering and Naive Bayes - we can predict ideological outcome (liberal vs. conservative as defined below) of US Supreme Court cases with test accuracy (or out-of-bag score for bagged methods) of around 72%.  Random forest algorithms tended to achieve the best results, though other techniques were close behind.  The exception was the extremely poor performance of Naive Bayes, which we attribute to the non-parametric distributions of our entirely categorical variables.  A preliminary assessment shows that a more complex ensemble method where multiple models are probability-voted to predict case outcomes is not necessarily more effective than simply training models with case-centered data alone.  

## Project Background

We are deeply indebted to the source of our labeled data, http://supremecourtdatabase.org, which has complete, expertly coded and labeled data about US Spreme Court cases dating back to 1791.  We used the "modern" version of the database, with cases since 1946, since the Court functioned quite differently in earlier periods.  The modern database has two different versions, and we use both.  The case-centered database contains 8,893 cases (as rows), each of which is described in 53 columns.  The justice-centered database contains the same information, but each case is split into 5 to 9 rows, one for each justice who voted in the case (usually 9 of course, so there are 79,612 total rows).  The justice-centered database contains 8 extra columns with justice names and their votes, whether he or she wrote the majority opinion etc.

We selected ideological direction (i.e. "liberal" vs. "conservative" as detailed on pages 50-52 of the codebook, included in this repo) as our target for prediction since this is generally what people want to know.  Prior research by Katz, Bommarito and Blackman targeted case disposition, coded in the database as any of 11 categories but essentially indicating whether the Court decided to affirm or reverse a lower-court decision.  However, a significant number of decisions (15% of their dataset, which included pre-1946 cases), at least at the justice-centered level, were ambiguous. Ideological direction, in contrast, is "unspecifiable" in only 1.7% of modern cases and blank for only 0.4%.  Ideological direction has the additional advantage of being balanced almost 50/50% between liberal and conservative.

As the lengthy codebook shows, all of the columns are essentially categorical.  In choosing predictor variables for our models, we first left aside non-predictive columns with unique values for each case (name of case, identification numbers, date of case).  Next we needed to eliminate any columns related to the outcome of a case.  This is because, in a genuine prediction scenario, we would not have data for anything not known prior to the Court's decision's being made public.  Thus we do not train our models with even tangentially outcome-related columns, such as the authority cited for a decision (could be 7 different values, including "statutory construction" and "federal common law").

Another concern in choosing predictors was the curse of dimensionality (multi-value columns leading to sparser arrays of dummy variables).  We have a reasonably large sample of cases, but not enough to support machine learning of generalizable distinctions among, for example, 310 different values of "petitioner type" (such as "state department or agency" vs. "state commission, board, committee or authority").  We attempted to balance the usefulness of any given variable against the number of values it could take, in the end settling on 14 variables that expand to around 450 columns once one-hot encoded.  As overfitting did later become an issue, future iterations of these models should narrow down/optimize the number of predictors further.

## Methodology

Our initial strategy was the simpler of the two main approaches we took: we split the case-centered data into testing and training sets and tried various machine learning techniques to predict the ideological direction of the full Court's decisions.  Individual justice identities (except for who was chief justice at the time) and vote counts were not used.  Results were encouraging: 70% or so test accuracy (75% or so ROC-AUC), but we wanted to make use of the justice centered data.

To do so we split the justice-centered data by justice, and then split each justice's cases into testing and training sets.  We then trained models for each of the 37 justices, tuning hyper-parameters for each using grid search.  The justice-centered models had anywhere from 5,087 rows for Brennan to 82 rows for Gorsuch.  These could be used as-is if individual-justice decisions are of interest, but we decided to use them as an ensemble to predict full-Court decisions.  For each case, we assemble a vector of probabilities for the justices who vote on that case (again, usually 9 but as few as 5 historically).  These are the predict_proba outputs of individual justice models for the case in question.  Using a Poisson binomial distribution (since the probabilities are uneven), we find the probability that a majority of the justices votes are liberal vs. conservative.  (This is computed by adding up the probability mass function for, say, 5, 6, 7, 8 or 9 justices' voting liberal if there are 9 justices sitting.  This is adjusted depending on how many justices are voting: the probability mass function is added for 4, 5 or 6 justices' voting liberal if only 6 justices are voting.)

We expected the ensemble method to improve on the initial strategy since the individual justices models' were individually tuned; but for now, any gain in accuracy, ROC-AUC and so forth has been minimal.  We plan to investigate further.

Note on data leakage: we had to be very careful to use the same test-train splits within each justice's model that we used for the case-centered approach.  That is, if a case was used for training in the case-centered approach, it had to be used only for training in each of the 9 or so justice-centered models where it was relevant.  Otherwise, if even one justice's model used the case for testing rather than training, the ensemble method would unfairly have "better" information about the test set than the case-centered strategy.  As a result, the test-train splits for each justice deviate randomly from the 70/30% split imposed on the case-centered data.  Another disadvantage of this necessary precaution was that it disallowed automatic k-folds cross validation as we needed to keep the same test-train split across all models.

## Python Pipeline

Please see AutoGenerateFull.py for the main script.

First of course, we import the databases, in both cases dropping rows where ideological direction is indeterminate.  We then drop columns not used as predictors or target, one-hot encode the remaining predictors and run a battery of machine-learning methods for the case-centered data, for each justice individually and then the ensemble method.

Records are generated in a text file, showing, for each model run, test and train accuracy, ROC-AUC and log-loss.  For the test data, precision, recall and confusion matrix are recorded.  For individual-justice models, test and train accuracy, ROC-AUC and log-loss are shown both with default scikit-learn settings an after tuning.  The optimal values for hyper-parameters are also noted.  These optimal values vary quite a bit by justice, but in general trees needed to be kept shallow (maximum depth of around 10) to avoid overfitting.

These records are reproduced in a CSV output file.  For each model we also export a CSV file with feature importances according to our random-forest classifier, which should be a rich source of further conclusions for this study.  

## Citations

Harold J. Spaeth, Lee Epstein, et al. 2018 Supreme Court Database, Version 2018 Release 1. URL: http://Supremecourtdatabase.org

Katz DM, Bommarito MJ, II, Blackman J (2017) A general approach for predicting the behavior of the Supreme Court of the United States. PLoS ONE 12(4): e0174698. https://doi.org/10.1371/journal.pone.0174698

Poisson binomial Python library: https://github.com/tsakim/poibin
