<!-- slide -->
#Decision Trees (Continued)

>I felt my lungs inflate with the onrush of scenery—air, mountains, trees, people. I thought, "This is what it is to be happy."
>
> --– **Sylvia Plath**

Reminder: you will need the `rpart` and `caret` packages.

<!-- slide -->

## Decision Trees

Decision trees are similar to k-nearest neighbors but instead of looking for neighbors, decision trees create neighborhoods. We won't explore the full details of trees, but just start to understand the basic concepts, as well as learn to fit them in R.

Neighborhoods are created via recursive binary partitions. In simpler terms, pick a feature and a possible cutoff value. Data that have a value less than the cutoff for the selected feature are in one neighborhood (the left) and data that have a value greater than the cutoff are in another (the right). Within these two neighborhoods, repeat this procedure until a stopping rule is satisfied. (More on that in a moment.) To make a prediction, check which neighborhood a new piece of data would belong to and predict the average of the $y_i$ values of data in that neighborhood.

<!-- slide -->

With the data from previous lecture, which has a single feature $x$, consider three possible cutoffs: -0.5, 0.0, and 0.75.

```{r, echo = FALSE}
plot_tree_split = function(cut = 0, main = "main") {
  plot(sim_slr_data, pch = 20, col = "grey", cex = 2, main = main)
  grid()
  abline(v = cut, col = "black", lwd = 2)
  left_pred = mean(sim_slr_data$y[sim_slr_data$x < cut])
  right_pred = mean(sim_slr_data$y[sim_slr_data$x > cut])
  segments(x0 = -2, y0 = left_pred, x1 = cut, y1 = left_pred, col = "limegreen", lwd = 2)
  segments(x0 = cut, y0 = right_pred, x1 = 2, y1 = right_pred, col = "firebrick", lwd = 2)
}
```

```{r, fig.height = 3, fig.width = 9, echo = FALSE}
par(mfrow = c(1, 3))
plot_tree_split(cut = -0.5, main = "cut @ x = -0.5")
plot_tree_split(cut = 0, main = "cut @ x = 0.0")
plot_tree_split(cut = 0.75, main = "cut @ x = 0.75")
```

<!-- slide -->

For each plot, the black vertical line defines the neighborhoods. The green horizontal lines are the average of the $y_i$ values for the points in the left neighborhood. The red horizontal lines are the average of the $y_i$ values for the points in the right neighborhood.

What makes a cutoff good? Large differences in the average $y_i$ between the two neighborhoods. More formally we want to find a cutoff value that minimizes

$$
\sum_{i \in N_L}(y_i - \hat{\mu}_{N_L}) ^ 2 + \sum_{i \in N_R}(y_i - \hat{\mu}_{N_R}) ^ 2
$$

where

- $N_L$ are the data in the left neighborhood
- $\hat{\mu}_{N_L}$ is the mean of the $y_i$ for data in the left neighborhood

<!-- slide -->

```{r, echo = FALSE}
calc_split_mse = function(cut = 0) {
  l_pred = mean(sim_slr_data$y[sim_slr_data$x < cut])
  r_pred = mean(sim_slr_data$y[sim_slr_data$x > cut])

  l_mse = round(sum((sim_slr_data$y[sim_slr_data$x < cut] - l_pred) ^ 2), 2)
  r_mse = round(sum((sim_slr_data$y[sim_slr_data$x > cut] - r_pred) ^ 2), 2)
  t_mse = round(l_mse + r_mse, 2)
  c("Total MSE" = t_mse, "Left MSE" = l_mse, "Right MSE" = r_mse)
}
```

```{r, echo = FALSE}
cbind(
  "Cutoff" = c(-0.5, 0.0, 0.75),
  rbind(
    calc_split_mse(cut = -0.5),
    calc_split_mse(cut = 0),
    calc_split_mse(cut = 0.75)
  )
) %>%
  kable() %>%
  kable_styling(full_width = FALSE)
```
<!-- slide -->


The table above summarizes the results of the three potential splits. We see that (of the splits considered, which are not exhaustive) the split based on a cutoff of $x = -0.50$ creates the best partitioning of the space.

<!-- slide -->

Now let's consider building a full tree.

```{r, echo = FALSE}
tree_slr = rpart(y ~ x, data = sim_slr_data)
```

```{r, echo = FALSE}
plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr, tibble(x = x)),
      col = "darkorange", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()
```

<!-- slide -->

In the plot above, the true regression function is the dashed black curve, and the solid orange curve is the estimated regression function using a decision tree. We see that there are two splits, which we can visualize as a tree.

```{r}
rpart.plot::rpart.plot(tree_slr)
```

The above "tree" shows the splits that were made. It informs us of the variable used, the cutoff value, and some summary of the resulting neighborhood. In "tree" terminology the resulting neighborhoods are "terminal nodes" of the tree. In contrast, "internal nodes" are neighborhoods that are created, but then further split.

<!-- slide -->


The "root node" or neighborhood before any splitting is at the top of the plot. We see that this node represents 100% of the data. The other number, 0.21, is the mean of the response variable, in this case, $y_i$.

Looking at a terminal node, for example the bottom left node, we see that 23% of the data is in this node. The average value of the $y_i$ in this node is -1, which can be seen in the plot above.

<!-- slide -->


We also see that the first split is based on the $x$ variable, and a cutoff of $x = -0.52$. (Note that because there is only one variable here, all splits are based on $x$, but in the future, we will have multiple features that can be split and neighborhoods will no longer be one-dimensional. However, this is hard to plot.)

<!-- slide -->

Let's build a bigger, more flexible tree.

```{r, echo = FALSE}
tree_slr = rpart(y ~ x, data = sim_slr_data, cp = 0, minsplit = 5)
```

```{r, echo = FALSE}
plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr, tibble(x = x)),
      col = "darkorange", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()
```

```{r, echo = FALSE}
rpart.plot::rpart.plot(tree_slr)
```

<!-- slide -->

There are two tuning parameters at play here which we will call by their names in R which we will see soon:

- `cp` or the "complexity parameter" as it is called. (Flexibility parameter would be a better name.) This parameter determines which splits are considered. A split must improve the performance of the tree by more than `cp` in order to be considered. When we get to R, we will see that the default value is 0.1.
- `minsplit`, the minimum number of observations in a node (neighborhood) in order to split again.

There are actually many more possible tuning parameters for trees, possibly differing depending on who wrote the code you're using. We will limit discussion to these two. Note that they effect each other, and they effect other parameters which we are not discussing. The main takeaway should be how they effect model flexibility.

<!-- slide -->

First let's look at what happens for a fixed `minsplit` by variable `cp`.

```{r, echo = FALSE}
tree_slr_cp_10 = rpart(y ~ x, data = sim_slr_data, cp = 0.10, minsplit = 2)
tree_slr_cp_05 = rpart(y ~ x, data = sim_slr_data, cp = 0.05, minsplit = 2)
tree_slr_cp_00 = rpart(y ~ x, data = sim_slr_data, cp = 0.00, minsplit = 2)
```

<!-- slide -->

```{r, fig.height = 4, fig.width = 12, echo = FALSE}
par(mfrow = c(1, 3))

plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "cp = 0.10, minsplit = 2")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr_cp_10, tibble(x = x)),
      col = "firebrick", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()

plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "cp = 0.05, minsplit = 2")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr_cp_05, tibble(x = x)),
      col = "dodgerblue", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()

plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "cp = 0.00, minsplit = 2")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr_cp_00, tibble(x = x)),
      col = "limegreen", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()
```

<!-- slide -->

We see that as `cp` *decreases*, model flexibility **increases**.

Now the reverse, fix `cp` and vary `minsplit`.

```{r, echo = FALSE}
tree_slr_ms_25 = rpart(y ~ x, data = sim_slr_data, cp = 0.01, minsplit = 25)
tree_slr_ms_10 = rpart(y ~ x, data = sim_slr_data, cp = 0.01, minsplit = 10)
tree_slr_ms_02 = rpart(y ~ x, data = sim_slr_data, cp = 0.01, minsplit = 02)
```

<!-- slide -->

```{r, fig.height = 4, fig.width = 12, echo = FALSE}
par(mfrow = c(1, 3))

plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "cp = 0.01, minsplit = 25")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr_ms_25, tibble(x = x)),
      col = "firebrick", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()

plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "cp = 0.01, minsplit = 10")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr_ms_10, tibble(x = x)),
      col = "dodgerblue", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()

plot(sim_slr_data, pch = 20, col = "grey", cex = 2,
     main = "cp = 0.01, minsplit = 2")
curve(cubic_mean(x), add = TRUE, lwd = 2, lty = 2)
curve(predict(tree_slr_ms_02, tibble(x = x)),
      col = "limegreen", lwd = 2, lty = 1, add = TRUE, n = 10000)
grid()
```

We see that as `minsplit` *decreases*, model flexibility **increases**.

<!-- slide -->


## Example: Credit Card Data

Let's use credit card data from the previous a built-in package. While last time we used the data to inform a bit of analysis, this time we will simply use this dataset to illustrate some concepts.

```{r}
# load data, coerce to tibble
crdt = as_tibble(ISLR::Credit)
```

<!-- slide -->

We are using the `Credit` data form the `ISLR` package. Note: **this is not real data.** It has been simulated.

```{r}
# data prep
crdt = crdt %>%
  select(-ID) %>%
  select(-Rating, everything())
```

We remove the `ID` variable as it should have no predictive power. We also move the `Rating` variable to the last column with a clever `dplyr` trick. This is in no way necessary, but is useful in creating some plots.

```{r}
# test-train split
set.seed(1)
crdt_trn_idx = sample(nrow(crdt), size = 0.8 * nrow(crdt))
crdt_trn = crdt[crdt_trn_idx, ]
crdt_tst = crdt[-crdt_trn_idx, ]
```

```{r}
# estimation-validation split
crdt_est_idx = sample(nrow(crdt_trn), size = 0.8 * nrow(crdt_trn))
crdt_est = crdt_trn[crdt_est_idx, ]
crdt_val = crdt_trn[-crdt_est_idx, ]
```
<!-- slide -->


After train-test and estimation-validation splitting the data, we look at the train data.

```{r}
# check data
head(crdt_trn, n = 10)
```

Recall that we would like to predict the `Rating` variable. This time, let's try to use only demographic information as predictors. In particular, let's focus on `Age` (numeric), `Gender` (categorical), and `Student` (categorical).

Let's fit KNN models with these features, and various values of $k$. To do so, we use the `knnreg()` function from the `caret` package. (There are many other KNN functions in R. However, the operation and syntax of `knnreg()` better matches other functions we will use in this course.) Use `?knnreg` for documentation and details.

```{r}
crdt_knn_01 = knnreg(Rating ~ Age + Gender + Student, data = crdt_est, k = 1)
crdt_knn_10 = knnreg(Rating ~ Age + Gender + Student, data = crdt_est, k = 10)
crdt_knn_25 = knnreg(Rating ~ Age + Gender + Student, data = crdt_est, k = 25)
```

<!-- slide -->

Here, we fit three models to the estimation data. We supply the variables that will be used as features as we would with `lm()`. We also specify how many neighbors to consider via the `k` argument.

But wait a second, what is the distance from non-student to student? From male to female? In other words, how does KNN handle categorical variables? It doesn't! Like `lm()` it creates dummy variables under the hood.

**Note:** To this point, and until we specify otherwise, we will always coerce categorical variables to be factor variables in R. We will then let modeling functions such as `lm()` or `knnreg()` deal with the creation of dummy variables internally.

<!-- slide -->

```{r}
head(crdt_knn_10$learn$X)
```

Once these dummy variables have been created, we have a numeric $X$ matrix, which makes distance calculations easy. For example, the distance between the 4th and 5th observation here is `r round(dist(head(crdt_knn_10$learn$X))[10], 2)`.

```{r}
dist(head(crdt_knn_10$learn$X))
```

```{r}
sqrt(sum((crdt_knn_10$learn$X[4, ] - crdt_knn_10$learn$X[5, ]) ^ 2))
```

(What about interactions? Basically, you'd have to create them the same way as you do for linear models. We only mention this to contrast with trees in a bit.)

<!-- slide -->

OK, so of these three models, which one performs best? (Where for now, "best" is obtaining the lowest validation RMSE.)

First, note that we return to the `predict()` function as we did with `lm()`.

```{r}
predict(crdt_knn_10, crdt_val[1:5, ])
```

This uses the 10-NN (10 nearest neighbors) model to make predictions (estimate the regression function) given the first five observations of the validation data. (**Note:** We did not name the second argument to `predict()`. Again, you've been warned.)

<!-- slide -->

Now that we know we already know how to use the `predict()` function, let's calculate the validation RMSE for each of these models.

```{r}
knn_mod_list = list(
  crdt_knn_01 = knnreg(Rating ~ Age + Gender + Student, data = crdt_est, k = 1),
  crdt_knn_10 = knnreg(Rating ~ Age + Gender + Student, data = crdt_est, k = 10),
  crdt_knn_25 = knnreg(Rating ~ Age + Gender + Student, data = crdt_est, k = 25)
)
```

```{r}
knn_val_pred = map(knn_mod_list, predict, crdt_val)
```

```{r}
calc_rmse = function(actual, predicted) {
  sqrt(mean((actual - predicted) ^ 2))
}
```

```{r}
map_dbl(knn_val_pred, calc_rmse, crdt_val$Rating)
```

<!-- slide -->

So, of these three values of $k$, the model with $k = 25$ achieves the lowest validation RMSE.

This process, fitting a number of models with different values of the *tuning parameter*, in this case $k$, and then finding the "best" tuning parameter value based on performance on the validation data is called **tuning**. In practice, we would likely consider more values of $k$, but this should illustrate the point.

<!-- slide -->

In the coming weeks, we will discuss the details of model flexibility and model tuning, and how these concepts are tied together. However, even though we will present some theory behind this relationship, in practice, **you must tune and validate your models**. There is no theory that will inform you ahead of tuning and validation which model will be the best. By teaching you *how* to fit KNN models in R and how to calculate validation RMSE, you already have all the tools you need to find a good model.

<!-- slide -->

Let's turn to decision trees which we will fit with the `rpart()` function from the `rpart` package. Use `?rpart` and `?rpart.control` for documentation and details.

We'll start by using default tuning parameters.

```{r}
crdt_tree = rpart(Rating ~ Age + Gender + Student, data = crdt_est)
```

```{r}
crdt_tree
```

Above we see the resulting tree printed, however, this is difficult to read. Instead, we use the `rpart.plot()` function from the `rpart.plot` package to better visualize the tree.

```{r}
rpart.plot(crdt_tree)
```

<!-- slide -->

At each split, the variable used to split is listed together with a condition. (If the condition is true for a data point, send it to the left neighborhood.) Although the `Gender` variable was used, we only see splits based on `Age` and `Student`. (This hints at the relative importance of these variables for prediction. More on this much later.)

Categorical variables are split based on potential categories! This is **excellent**. This means that trees naturally handle categorical features without needing to convert to numeric under the hood. We see a split that puts students into one neighborhood, and non-students into another.

Notice that the splits happen in order. So for example, the third terminal node (with an average rating of 298) is based on splits of:

- `Age < 83`
- `Age < 70`
- `Age > 39`
- `Student = Yes`

In other words, individuals in this terminal node are students who are between the ages of 39 and 70. (Only 5% of the data is represented here.) This is basically an interaction between `Age` and `Student` without any need to directly specify it! What a great feature of trees.
<!-- slide -->

To recap:

- Trees do not make assumptions about the form of the regression function.
- Trees automatically handle categorical features.
- Trees naturally incorporate interaction.

Now let's fit another tree that is more flexible by relaxing some tuning parameters. (By default, `cp = 0.1` and `minsplit = 20`.)

```{r}
crdt_tree_big = rpart(Rating ~ Age + Gender + Student, data = crdt_est,
                      cp = 0.0, minsplit = 20)
```

```{r}
rpart.plot(crdt_tree_big)
```
<!-- slide -->

To make the tree even bigger, we could reduce `minsplit`, but in practice we mostly consider the `cp` parameter. (We will occasionally modify the `minsplit` parameter on quizzes.) Since `minsplit` has been kept the same, but `cp` was reduced, we see the same splits as the smaller tree, but many additional splits.

Now let's fit a bunch of trees, with different values of `cp`, for tuning.

```{r}
tree_mod_list = list(
  crdt_tree_0000 = rpart(Rating ~ Age + Gender + Student, data = crdt_est, cp = 0.000),
  crdt_tree_0001 = rpart(Rating ~ Age + Gender + Student, data = crdt_est, cp = 0.001),
  crdt_tree_0010 = rpart(Rating ~ Age + Gender + Student, data = crdt_est, cp = 0.010),
  crdt_tree_0100 = rpart(Rating ~ Age + Gender + Student, data = crdt_est, cp = 0.100)
)
```

```{r}
tree_val_pred = map(tree_mod_list, predict, crdt_val)
```

```{r}
map_dbl(tree_val_pred, calc_rmse, crdt_val$Rating)
```

<!-- slide -->

Here we see the least flexible model, with `cp = 0.100`, performs best.

Note that by only using these three features, we are severely limiting our models performance. Let's quickly assess using all available predictors.

```{r}
crdt_tree_all = rpart(Rating ~ ., data = crdt_est)
```

```{r}
rpart.plot(crdt_tree_all)
```

Notice that this model **only** splits based on `Limit` despite using all features. (This should be a big hint about which variables are useful for prediction.)

<!-- slide -->

```{r}
calc_rmse(
  actual = crdt_val$Rating,
  predicted = predict(crdt_tree_all, crdt_val)
)
```

This model performs much better. You should try something similar with the KNN models above. Also, consider comparing this result to results from last chapter using linear models.

Notice that we've been using that trusty `predict()` function here again.

```{r}
predict(crdt_tree_all, crdt_val[1:5, ])
```

What does this code do? It estimates the mean `Rating` given the feature information (the "x" values) from the first five observations from the validation data using a decision tree model with default tuning parameters. Hopefully a theme is emerging.
