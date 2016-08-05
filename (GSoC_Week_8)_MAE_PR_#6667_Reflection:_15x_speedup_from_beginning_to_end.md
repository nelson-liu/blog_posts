If you've been following this blog, you'll notice that I've been talking a lot about the [weighted median problem](http://blog.nelsonliu.me/2016/07/05/gsoc-week-6-efficient-calculation-of-weighted-medians/), as it is intricately related to optimizing the mean absolute error (MAE) impurity criterion. The scikit-learn pull request I was working on to add aforementioned criterion to the [`DecisionTreeRegressor`](http://scikit-learn.org/stable/modules/generated/sklearn.tree.DecisionTreeRegressor.html), [PR #6667](https://github.com/scikit-learn/scikit-learn/pull/6667), has received approval from several reviewers for merging. Now that the work for this PR is complete, I figure that it's an apt time to present a narrative of the many iterations it took to converge to our current solution for the problem.

# Iteration One: Naive Sorting
The `Criterion` object that is the superclass of MAE has a variety of responsibilities during the process of decision tree construction, primarily  evaluating the impurity of the current node, and evaluating the impurity of all the possible children to find the best next split. In the first iteration, every time we wanted to calculate the impurity of a set of samples (either a node, or a possible child), we would sort this set of samples and extract the median from it.
After implementing this, I ran some benchmarks to see how fast it was compared to the Mean Squared Error (MSE) criterion currently implemented in the library. I used both the classic Boston housing price dataset and a larger, synthetic dataset with 1000 samples and 100 features each to compare. Training was done on 0.75 of the total dataset, and the other 0.25 was used as a held-out test set for evaluation.

### Boston Housing Dataset Benchmarks: Iter. 1
```
MSE time: 105 function calls in 0.004 seconds
MAE time:  105 function calls in 0.175 seconds

Mean Squared Error of Tree Trained w/ MSE Criterion: 32.257480315
Mean Squared Error of Tree Trained w/ MAE Criterion: 29.117480315

Mean Absolute Error of Tree Trained w/ MSE Criterion: 3.50551181102
Mean Absolute Error of Tree Trained w/ MAE Criterion: 3.36220472441
```

### Synthetic Dataset Benchmarks: Iter. 1
```
MSE time: 105 function calls in 0.089 seconds
MAE time:  105 function calls in 15.419 seconds

Mean Squared Error of Tree Trained w/ MSE Criterion: 0.702881265958
Mean Squared Error of Tree Trained w/ MAE Criterion: 0.66665916831

Mean Absolute Error of Tree Trained w/ MSE Criterion: 0.650976429446
Mean Absolute Error of Tree Trained w/ MAE Criterion: 0.657671579992
```

This sounds reasonable enough, but we quickly discovered after looking at the numbers that it was intractable; while sorting is quite fast in general, sorting in the process of finding the children was completely unrealistic. For a sample set of size `n`, we would divide it into `n-1` partitions of left and right child, and sort each one, on every node. The larger dataset made MSE take 22.25x more time, but it made MAE take 88.11x (!) slower. This result was obviously unacceptable, so we began thinking of how to optimize; this led us to our second development iteration.

# Iteration 2: MinHeap to Calculate Weighted Median
In iteration two, we implemented the algorithm / methodology I discussed in my [week 6 blog post](http://blog.nelsonliu.me/2016/07/05/gsoc-week-6-efficient-calculation-of-weighted-medians/). With the method, we did away with the time associated with sorting every sample set for every node and possible child and instead "saved" sorts, using a modified bubblesort to insert and remove new elements from the left and right child heaps efficiently. This algorithm had a substantial impact on the code --- rerunning the benchmarks we used earlier yielded the following results (MSE results remained largely the same due to run-by-run variation, but accuracy is the same as is thus omitted):

### Boston Housing Dataset Benchmarks: Iter. 2
```
MSE time: 105 function calls in 0.004s (was: 0.004s)
MAE time:  105 function calls in 0.276s (was: 0.175s)
```

### Synthetic Dataset Benchmarks: Iter. 2
```
MSE time: 105 function calls in 0.065s (was: 0.089s)
MAE time:  105 function calls in 5.952s (was: 15.419s)
```
After this iteration, MAE is still quite slower than MSE, but it's a definite improvement from naive sorting (especially when using a large dataset). I found it interesting that the new method is actually a little bit slower than the naive method we first implemented on the relatively small Boston dataset (0.276s vs 0.175s, respectively). My mentors and I hypothesized that this might be due to the time cost associated with creating the `WeightedMedianCalculators` (the objects that handled the new median calculation), though their efficiency in calculation is supported by the speed increase from 15.419s to 5.952s on the larger randomly generated dataset. 5.952 seconds on a dataset with 1000 samples is still slow though, so we kept going. 
# Iteration 3: Pre-allocation of objects
We suspected that there could be a high cost associated with spinning up objects used to calculate the weighted median. This is especially important because the majority of the `tree` code in scikit-learn is written in Cython, which disallows us of Python objects and functions. This is because we run the Cython code without the Python GIL (global interpreter lock). The GIL is a mutex that prevents multiple native threads from executing Python bytecodes at once, so running without the GIL makes our code a lot faster. However, because our `WeightedMedianCalculator`s are Python objects, we unfortunately need to reacquire the GIL to instantiate them. We predicted that this could be a major source of the bottleneck. As a result, I implemented a `reset` function in the objects to clear them back to their state at construction, which could be executed without the GIL. When we first ran the C-level constructor (it is run at every node, as opposed to the Python constructor that is run only once), we evaluated whether the `WeightedMedianCalculator`s had been created or not; if they have not been, we reacquire the GIL and create them. If they have, we simply reset them. This allowed us to only reacquire the GIL once throughout the algorithm, which, as predicted, led to substantial speedups. Running the benchmarks again displayed:
### Boston Housing Dataset Benchmarks: Iter. 3
```
MSE time: 105 function calls in 0.009s (was: 0.004s, 0.004s)
MAE time:  105 function calls in 0.038s (was: 0.276s, 0.175s)
```

### Synthetic Dataset Benchmarks: Iter. 3
```
MSE time: 105 function calls in 0.065s (was: 0.065s, 0.089s)
MAE time:  105 function calls in 0.978s (was: 5.952s, 15.419s)
```
Based on the speed improvement from the most recent changes, it's reasonable to conclude that a large amount of time was spent re-acquiring the GIL. With this approach, we cut down the time spent reacquiring the GIL by quite a significant amount since we only need to do it once, but ideally we'd like to do it zero times. This led us to our third iteration.

# Iteration 4: Never Re-acquire the GIL
Constructing the `WeightedMedianCalculator`s requires two pieces of information, `n_outputs` (the number of outputs to predict) and `n_node_samples` (the number of samples in this node). We need to create a `WeightedMedianCalculator` for each output to predict, and the internal size of each should be equal to `n_node_samples`. 
We first considered whether we could allocate the `WeightedMedianCalculator`s at the `Splitter` level (the splitter is in charge of finding the best splits, and uses the `Criterion` to do so). In `splitter.pyx`, the `__cinit__` function (Python-level constructor) only exposes the value of `n_node_samples` and we lack the value of `n_outputs`. The opposite case is true in `criterion.pyx`, where the `__cinit__` function is only shown the value of `n_outputs` and does not get `n_node_samples` until C-level init time, hence why we previously were constructing the WeightedMedianHeaps in the init function and cannot completely do it in `__cinit__`. If we could do it completely in the `__cinit__`, we would not have to reacquire the GIL because the `__cinit__` operates on the Python level in the first place.
As a result, we simply modified the `__cinit__` of the `Criterion` objects to expose the value of `n_node_samples`, allowing us to do all of the allocation of the objects at the Python-level without having to specifically reacquire the GIL. We reran the benchmarks on this, and saw minor improvements in the results:
### Boston Housing Dataset Benchmarks: Iter. 4
```
MSE time: 105 function calls in 0.003s (was: 0.009s, 0.004s, 0.004s)
MAE time:  105 function calls in 0.032s (was: 0.038s, 0.276s, 0.175s)
```

### Synthetic Dataset Benchmarks: Iter. 4
```
MSE time: 105 function calls in 0.065s (was: 0.065s, 0.065s, 0.089s)
MAE time:  105 function calls in 0.961s (was: 0.978s, 5.952s, 15.419s)
```

# Conclusion
So after these four iterations, we managed to get a respectable 15x speed improvement. There's still a lot of work to be done, especially with regards to speed on larger datasets; however, as my mentor Jacob commented, "Perfect is the enemy of good", and those enhancements will come in future (very near future) pull requests.


If you have any questions, comments, or suggestions, you're welcome to leave a comment below.

Thanks to my mentors [Raghav RV](https://github.com/raghavrv) and [Jacob Schreiber](https://github.com/jmschrei) for their input on this problem; we've run through several solutions together, and they are always quick to point out errors and suggest improvements.

_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu)  if you want to track the progress of my Summer of Code project, or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
