The [scikit-learn](https://github.com/scikit-learn/scikit-learn) [pull request](https://github.com/scikit-learn/scikit-learn/pull/6954) I opened to add impurity-based pre-pruning to `DecisionTree`s and the classes that use them (e.g. the `RandomForest`, `ExtraTrees`, and `GradientBoosting` ensemble regressors and classifiers) was merged a week ago, so I figure that this would be an appropriate place to talk about what this actually does and provide an example of it in action.

# Decision Tree Node Impurity - A Recap
_Note: if you're familiar with what the "impurity" of a node in a decision tree is, feel free to skip this_

In decision tree-based classification and regression methods, the goal is to iteratively split to minimize the "impurity" of the partitioned dataset (see [my week 2 blog post](http://blog.nelsonliu.me/2016/06/06/gsoc-week-2-intro-to-decision-trees/) for more details about this). The definition of node impurity varies based on the method used to calculate it, but in rough terms it measures how "pure" a leaf node is. If a leaf node contains samples that all belong to one class (for classification) or have the same real-valued output (for regression), it is "pure" and thus has an impurity of 0.

The ultimate goal of decision tree-based models is to split the tree such that each leaf node corresponds to the prediction of a single class, even if there is only one sample in that class. However, this can lead to the tree radically overfitting the data; it will grow in a manner such that it will create a leaf node for every sample if necessary. Overfitting is when the decision tree continues to grow and reduce the training set error, but at the expense of teh test set error. In other words, it can basically memorize the samples of the train set, and may lose the ability to generalize well to new datasets. One method for avoiding overfitting in decision trees is **pre-pruning**.

In **pre-pruning**, you stop the decision tree growth before it perfectly fits the training data; this is because (as outlined in the previous paragraph) fitting the training data perfectly often leads to overfitting.

The [scikit-learn `tree module`](http://scikit-learn.org/stable/modules/classes.html#module-sklearn.tree), there are a variety of methods used during tree growth to decide whether a node should be split or whether it should be declared a leaf node.

- Is the current depth of the tree greater than the user-set `max_depth` parameter? `max_depth` is the maximum depth of the tree. If `None`, then nodes are expanded until all leaves are pure or until all leaves contain less than `min_samples_split` samples. `(depth >= max_depth) `

- Or is the number of samples in this current node less than the value of `min_samples_split`? `min_samples_split` is the minimum number of samples required to split an internal node. `(n_node_samples < min_samples_split)`

- Or is the number of samples in this current node less than `2 * min_samples_leaf`? `min_samples_leaf` is the minimum number of samples required to be in a leaf node.
`(n_node_samples < 2 * min_samples_leaf)`

-  Or is the total weight of all of the samples in the node less than `min_weight_leaf`? `min_weight_leaf` defines the minimum weight required to be in a leaf node. `(weighted_n_node_samples < min_weight_leaf)`

- Or lastly, is the impurity of the node equal to 0?

By changing the value of each of these constructor parameters, it's possible to achieve a pseudo-prepruning effect. For example, setting the value of `min_samples_leaf` can define that each leaf has more than one element, thus ensuring that the tree cannot perfectly (over)fit the training dataset by creating a bunch of small branches exclusively for one sample each. In reality, what this is actually doing is simply just telling the tree that each leaf doesn't HAVE to have an impurity of 0.

# Enter `min_impurity_split`
My contribution was to create a new constructor parameter, `min_impurity_split`; this value defines the minimum impurity that the node must have in order to not be a leaf. For example, if the user-defined value of `min_impurity_split` is `0.5`, a node with an impurity of `0.7` would be further split on but nodes with impurities of `0.5` and `0.2` would be declared leaves and receive no further splitting. In this manner, it's possible to control the grain to which the decision tree fits the data, allowing for coarser fits if desired.

# This is great and all, but does it improve my estimator's performance?

`min_impurity_split` helps to control over-fitting and as a result can improve your estimator's performance if it is overfitting on the dataset. I ran a few benchmarks in which I plotted the number of nodes in trees fit with a variety of parameters, and their performance on a held-out test set on the Boston housing prices regression task.

![Fitted Tree Number of Nodes vs. Parameters](/content/images/2016/08/num_nodes_given_parameters.png)

![Prediction Mean Squared Error vs. Paramters](/content/images/2016/08/MSE_given_parameters.png)

_Note: in the chart on the bottom, the y-axis represents the Mean Squared Error (MSE) -- thus, lower error is better._

These graphs show some interesting things that I want to point out. If you look at the first chart, you'll see that the tree grown with default parameters is massive, with over 700 nodes. With large trees, it's generally quite easy to overfit since it indicates that the tree is maybe creating leaf nodes for individual samples, which is generally detrimental for generalization. On the other hand, the tree grown with `min_impurity_split = 3` is much more modest, with ~150 nodes. If you examine the graph, you'll see that as the value of `min_impurity_split` decreases, the number of nodes increases which makes sense.

Looking at the second chart, you will see that setting various values of `min_impurity_split` and other parameters generally serve to improve the performance of the estimator by reducing MSE. These parameters all limit tree growth in some way. There are some notable exemptions though --- for example, note that the tree grown with `max_depth = 3` has a relatively high MSE. This is because, referencing the first chart, it is tiny! As a result, this tree could have been under-fit. Thus, it's important to maintain a balance in tree size. Trees with too many nodes overfit easily, but you don't want to prune so much that you render it incapable of fitting in general!

If you have any questions, comments, or suggestions, you're welcome to leave a comment below.

Per usual, thanks goes out to my mentors [Raghav RV](https://github.com/raghavrv) and [Jacob Schreiber](https://github.com/jmschrei) for taking a look at the PR and reviewing my code.

_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu)  if you want to track the progress of my Summer of Code project, or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
