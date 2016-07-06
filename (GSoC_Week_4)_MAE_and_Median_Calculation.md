In the first part of my project, I am implementing the [Mean Absolute Error](https://github.com/scikit-learn/scikit-learn/pull/6667) criterion for the scikit-learn [DecisionTreeRegressor](http://scikit-learn.org/stable/modules/generated/sklearn.tree.DecisionTreeRegressor.html). In this blog post, I'll talk about what the criterion does, as well as a technical / design challenge I faced while implementing it and two solutions I used.

# Criterion and impurity
When growing the tree from data, we want to find the best split at each possible level in the tree. To do so, we minimize the impurity (such that we want all of our nodes to be pure). A pure node in classification is one where all of the training samples are of a single class, and a pure node in regression is one where all of the training samples have the same value (for a review of how decision trees work, see my [week 2 post](http://blog.nelsonliu.me/2016/06/05/gsoc-week-2-intro-to-decision-trees/) or the plethora of resouces on the web). For the rest of the post, I'll be focusing on regression (as it is the domain of choice for my project).

There are a myriad of ways to define the node impurity, which can drastically affect the way the tree is grown. One of the most common impurity criterion is mean squared error (also known as sum of squared error, or MSE). When using mean squared error, the tree makes splits to optimize $$MSE = \frac{1}{n} \sum\_{i}^{n} (y\_{i} - f(x\_{i}))^2 $$

This formula might be a little abstract, so I'll go over what each of the terms means. In this case, \\(n \\) is the weighted number of samples in the current node (sum of all the weights). \\( x\_i \\) denotes the features of the sample \\( i \\). \\( y\_i \\) denotes the correct value of sample \\( i \\). \\( f(x\_i) \\) denotes the predicted value of sample \\( i \\) (think of the function \\( f \\) as taking in a sample and returning a prediction). As a result, the MSE sums up all of the square of the "errors" (difference between predicted and real values).

In MAE, we want to minimize 
$$ MAE = \frac{1}{n} \sum\_{i}^{n} |y\_i - f(x\_i)| $$ Where \\( y\_i \\) is again the true value and \\( f(x\_i) \\) is the predicted value. To minimize the MAE, you need to use the median of the \\( y\_i \\)s, due to the fact that you are forced to make the same prediction for all samples in a node.

As a result, the weighted median problem is a large part of this criterion, as we need to find the weighted median of the samples and use that as our \\( y\_i \\). In this blog post, we will temporarily ignore the weights and assume that all samples are equally weighted. There isn't much in the literature about efficient calculation of weighted medians, so my mentors and I are still working on methods to do it acceptably.

A common way to define the median of a set of samples is as the value such that the number of samples above and below the median are both less than or equal to half the total number of samples. The weighted median is thus a similar, but we seek to find a value such that the total weights of the samples above and below the median are both less than or equal to half the total weight of all samples. If this seems a bit strange, don't worry! Examples are provided a bit further below.

# Efficiently calculating the median
The speed of the mean absolute error criterion depends heavily on the implementation of the median calculation algorithm. In the rest of the post, I will go over two methods I used to calculate the median for any range of samples in a node --- one naive initial implementation, and an solution specifically optimized for the DecisionTreeRegressor.

More formally, the problem is as follows: Given an sorted array X denoting the samples and an array Y denoting the values corresponding to the samples in X, calculate the median of the Y's corresponding to X that cover the entire set. This is a bit confusing, so I will give an example below.

For example, define the samples `X = [[1], [3], [8], [7]]` and the values `Y = [3, 1, 6, 9]`. Define the `start` index as 0, and the `end` index as 4. We will use the variable `pos` to denote intermediary splits that divide the set of values in two the two subsets we need. The problem is to calculate the median of `samples[start:pos]` and `samples[pos:end]` quickly for where pos is the set of values between `start` and `end` (exclusive).

So in this case, the first iteration would be:
```
start = 0, pos = 1, end = 4:
Find the median of samples[start:pos]:
Y = [3]
Thus, the median of the values is 3.

Find the median of samples[pos:end]:
Y = [1, 6 , 9]
Thus, the median of the values is 6.
```

In the second iteration, we increment `pos` by one, and calculate the new medians.
```
start = 0, pos = 2, end = 4:
Find the median of samples[start:pos]:
Y = [3, 1]
Thus, the median of the values 2.

Find the median of samples[pos:end]:
Y = [6 , 9]
Thus, the median of the values is 7.5.
```

Moving on, you'd increment pos one last time to calculate the medians between `samples[0:3]` and `samples[3:4]`, giving you 3 and 9 (see if you can verify this for yourself!).

This problem may seem quite contrived and random, but it is exactly the process of finding the best split from a given node. The node needs to split at some value of `pos` between `start` and `end` that minimizes the impurity. As a result, it simply tries all values of `pos` between `start` and `end` to find the minimum and performs the split at that value of `pos`. The samples from `start` to `pos` is a candidate left child to split on, and the samples from `pos` to `end` is a candidate right child.

## Naive Implementation
Initially, I implemented an extremely naive version of the median calculation because I did not realize just how many splits it would be searching over. In the naive implementation, I put the values for each position of `pos` in a new array, and then sorted this new array to find the median. AS a result, I was sorting an array of variable size every time a candidate split was to be evaluated; this was incredibly inefficient.

In trying to refactor this, it was important to take into consideration the choice of data structure and the idea that re-sorting the whole array when shifting over `pos` by one would be wasteful. This eventually led me to the solution implemented in my project.

## MedianHeaps Implementation
After discussion with my mentors Jacob and Raghav, we realized that this problem was the same as finding the median of a running stream of numbers. As a result, I decided to implement a modified MedianHeap used to solve these sort of problems.

A heap is a datastructure that takes values and orders them such that the maximum (in a max heap) or minimum (in a min heap) is always at the top. Heaps are usually implemented internally as an array-represented tree, where the maximum or minimum value of all the data in the heap is the root node of the tree. The specific implementation details of the heap are widely documented, and is thus out of the scope of this blog post. I implemented a `MinMaxHeap` object in Cython, which could be used as either a min or max heap (based on a parameter passed in at construction). The `MinMaxHeap` had methods `is_empty()`(check if there are any elements in the heap), `size()` (return the number of elements in the heap), `push(DOUBLE_t)` (add a value to the heap), `remove(DOUBLE_t)` (remove a specific value from the heap), `pop(DOUBLE_t*)` (remove the top value from the heap and store it in a pointer of `DOUBLE_t`), and `peek(DOUBLE_t*)` (store the top value in the heap in a pointer of `DOUBLE_t`).

With this `MinMaxHeap` class, I was able to build another MedianHeap class. A `MedianHeap` uses a max heap and a min heap to efficiently calculate the median of a dataset at any given time, and allows quick insertion and deletion (because you don't have to re-sort the entire dataset on each modification). To understand how the `MedianHeap` works, it's useful to visualize the median as the center point of two halves of the data. On the left half, containing all the values less than the current median, is a max heap. On the right half, containing all the values greater than the current median, is a min heap. This configuration allows us to solve for the median quickly --- the median is at the top of whatever heap is larger, and is the average of the top of both heaps if they are the same size. Maintaining this nice property in median calculation comes with some extra complications in insertion and removal, though. When inserting a value into the `MedianNode`, it's important to pick the correct internal heap to add to (either the min heap on the right or the max heap on the left). If the value to be inserted is greater than the median, it goes in the right (and thus to the min heap). Similarly, if the value to be inserted is less than the median, it is inserted to the left (the max heap). After inserting, it's important to rebalance the `MedianHeap`; this involves moving values in the left to the right, or vice versa. This is important to ensure the property that the two `MinMaxHeap`s have a size within one of each other. Removal follows a similar procedure as addition (to decide which heap to remove from), and rebalancing is also needed afterwards.

To efficiently solve the problem outlined earlier, we use two `MedianHeaps` --- one for the left (`samples[start:pos]`) and one for the right (`samples[pos:end]`). Initially, we add all samples except one to the `MedianHeap` on the right, and calculate the medians accordingly. When moving `pos`, we don't have to res-sort and recalculate the median anymore! We can simply remove the value corresponding to the old value of `pos` from the right `MedianHeap`, and add it to the `MedianHeap` on the left. In this manner, we can efficiently calculate the impurity for all possible children nodes at split time.

Using two `MedianHeaps` in tandem resulted in a massive increase in speed for training --- more than 10x on large datasets! While MAE is still quite slow compared to MSE (mainly because there is no proxy function for calculating the median, thus at least some amount of sorting is needed at some time step), implementing a calculation scheme with `MedianHeap`s speeds it up enough and makes it viable for practical use.

If you have any questions, comments, or suggestions, you're welcome to leave a comment below :)

Thanks to my mentors [Raghav RV](https://github.com/raghavrv) and [Jacob Schreiber](https://github.com/jmschrei) for guiding me through any questions I have and helping me work through them.

_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu)  if you want to track the progress of my Summer of Code project, or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
