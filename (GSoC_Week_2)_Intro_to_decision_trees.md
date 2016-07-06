*Apologies for the late post, I had this sitting in my drafts and forgot to publish it!*

Decision Trees (DTs) are a non-parametric supervised learning method used for classification and regression popular for their ease of use and interpretability.

## Overarching Goals
In building a decision tree from data, the goal is to create a model that predicts the value of a target variable by learning simple binary "decision rules".

For example, here is a decision tree used to classify the species of a variety of iris given data.

![Iris Decision Tree. Image by scikit-learn](http://scikit-learn.org/stable/_images/iris.svg)

In this image, the rules for splitting are in the non-terminal nodes of the tree. For example, starting at the top of the tree: if a given sample has a petal length (in cm) ≤ 2.45, it is classified as being of species _setosa_. If the sample has a petal length of more than 2.45, we look at more rules to classify it (the right child of the root node). The tree is simple to interpret, and works surprisingly well.

## Building the Tree
The process of constructing a tree is a bit more involved, but quite intuitive. The gist of the problem is deciding which sample attribute to split on, and what value of the attribute to split on.

Imagine the you have a set of (x,y) coordinates, and each of these coordinates has an associated label that you wish to predict (say, red or blue).
![Decision Tree Boundary Visualization](/content/images/2016/05/decision_tree_boundaries.png)

Your task is to draw straight lines (parallel to either the x or y-axis) to divide or "partition" the data space. A reasonable first cut would be to draw the line x = 0.5 (the black line in the image above), dividing the space in half. In these halves, the right side is composed entirely of blue points. As a result, this part of the space is "pure" (composed of data points of only one class).

Moving forwards, we turn our attention to the left half of the space. A next reasonable split would be the line y=0.3 (approximately). This split, visualized by the yellow line, once again produces a set of pure points in 0 ≤ x ≤ 0.5 and 0 ≤ y ≤ 0.3. We can also split on the purple line (x = .1), thus dividing all of the points of various classes into their separate partitions. 

By doing so, we have constructed the decision rules necessary to learn our tree. By iteratively splitting, we can narrow down the space on the basis of x and y coordinates until we determine the proper class. The decision tree constructed is shown below.

![Simple Decision Tree](/content/images/2016/05/simple_decision_tree.png)

This example is simple enough when using data with one two attributes (e.g. x and y), but things quickly get tricky as the number of dimensions increases. To calculate the best splits, decision trees use various criterion of error that are essentially different metrics of "good" splits. The first part of my GSoC project will be implementing the mean absolute error (MAE) criterion, which judges potential splits by their absolute distance away from the true value.

I hope this blog post was informative and clear in describing what decision trees are, as well as the iterative partitioning process of growing trees. If you want to learn more, Pedro Domingos has an informative set of lecture videos on decision trees in [~~his Coursera course (see week 2)~~](https://class.coursera.org/machlearning-001/lecture) (EDIT: Coursera has unfortunately removed the course from their catalog. Please let me know if you find a good video resource that explains decision trees in an intuitive and clear way).

_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu)  if you want to track the progress of my Summer of Code project, or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
