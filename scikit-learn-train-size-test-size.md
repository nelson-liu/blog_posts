I recently authored a [scikit-learn PR](https://github.com/scikit-learn/scikit-learn/pull/7459) to edit the behavior of `train_size` and `test_size` in most of the classes that use it; I thought that their interaction was simple and obvious, but was recently informed otherwise. In this blog post, I'll go through the problem as well as the coming fixes.

TL;DR: Simply setting `test_size=some_value` may not give the same results as setting `train_size=1-some_value`.

# `test_size` and `train_size`
If you're a user of the [scikit-learn](http://scikit-learn.org/stable/) Python machine learning library, you might know that many of the cross validation classes (e.g. [`model_selection.StratifiedShuffleSplit`](http://scikit-learn.org/stable/modules/generated/sklearn.model_selection.StratifiedShuffleSplit.html) and [`model_selection.KFold`](http://scikit-learn.org/stable/modules/generated/sklearn.model_selection.KFold.html)) and the popular [`model_selection.train_test_split`](http://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html) method take the `test_size` and `train_size` parameters to define the amount of data used in the "train" split and the amount used in the "test" split. If the parameters are floats, they represent the proportion of the dataset in the split; if they are ints, the represent the absolute number of samples in the split.

## `train_test_split` behavior
Here's a pop quiz for those familiar with the methods (if not, feel free to read through the code and try to reason why the answers are what they are). What happens when we execute the following code in scikit-learn 0.18? (though it should be the same for all recent versions):
```python
from __future__ import print_function
import numpy as np
from sklearn.model_selection import train_test_split
# X is a numpy array of "data" with length 10
X = np.zeros(10)

# y is a numpy array of "labels" with length 10 
# corresponding to the data
y = np.zeros(10)

X_train, X_test, y_train, y_test = train_test_split(X, y, 
                                                    test_size=0.3)
print(len(X_train), len(X_test), len(y_train), len(y_test))


X_train, X_test, y_train, y_test = train_test_split(X, y, 
                                                    train_size=0.7)
print(len(X_train), len(X_test), len(y_train), len(y_test))


X_train, X_test, y_train, y_test = train_test_split(X, y, 
                                                    train_size=0.3, 
                                                    test_size=0.5)
print(len(X_train), len(X_test), len(y_train), len(y_test))
```

The answers are:
```python
7 3 7 3
7 3 7 3
3 5 3 5
```

These responses all seem pretty intuitive; the last example isn't used too often, but you're basically sub-sampling the dataset. If `test_size + train_size > 1.0`, scikit-learn will complain.

## cross-validation behavior
Try this one --- what happens when we execute the following code?
```python
from __future__ import print_function
import numpy as np
from sklearn.model_selection import StratifiedShuffleSplit

# X is a numpy array of "data" with length 10
X = np.zeros(10)
# y is a numpy array of "labels" with length 10 
# corresponding to the data
y = np.zeros(10)

sss = StratifiedShuffleSplit(n_splits=1, test_size=0.3)
for train, test in sss.split(X,y):
    print(len(train), len(test)) # case one

sss = StratifiedShuffleSplit(n_splits=1, train_size=0.7)
for train, test in sss.split(X,y):
    print(len(train), len(test))  # case two

sss = StratifiedShuffleSplit(n_splits=1, train_size=0.3, 
                             test_size=0.5)
for train, test in sss.split(X,y):
    print(len(train), len(test)) # case three
```

The answers in this case are:
```
7 3
7 1
3 5
```

# Wait, what? Why?
This isn't intuitive for a lot of people --- why does `StratifiedShuffleSplit` (and other cross-validation classes) give different results when `test_size = 1 - train_size`? The answer is pretty nuanced, and I'll try to explain it here.

In [`StratifiedShuffleSplit`](http://scikit-learn.org/0.18/modules/generated/sklearn.model_selection.StratifiedShuffleSplit.html), note that the default value for `test_size=0.1` and the default value for `train_size=None`. So when you set `test_size` but not `train_size` (as in the first case above), you've got `test_size=your setting` and `train_size=None`. Since `test_size` is set, scikit-learn just sets your `train_size = 1 - test_size`, as many would expect.

Let's jump ahead to case three for a second -- `train_size` and `test_size` sum to values less than one, but that's ok! The library just takes that as a signal that you want to subsample, and it'll obediently return training splits with the size `train_size` and test splits with the size `test_size`, leaving some of the data unused.

Now that you know that behavior, case two should make (some) sense. You explicitly set `train_size=0.7`, but leave `test_size` default. However, since `test_size=0.1` by default, your params are actually `train_size=0.7, test_size=0.1`. Thus, you end up subsampling your data and not using it all! The CV behavior in this case differs from `train_test_split`, since `test_size=None, train_size=None` by default there so you don't end up subsampling your data if you just use `train_size` in `train_test_split`.

# Coming changes
I've commonly heard the following from folks teaching others how to use scikit-learn, mostly in the context of `train_test_split`:
> Oh yeah it doesn't matter if you use `train_size` or `test_size`, it'll automatically set the other to the complement of the one you specify

While this is true for `train_test_split`, it's a dangerous thing to think in general --- or it was. After this was raised as a bug report in [scikit-learn/scikit-learn#5940](https://github.com/scikit-learn/scikit-learn/issues/5940), the devs decided to change the unintuitive behavior of setting `train_size` but not `test_size`. Starting from `sklearn 0.19`, if you use `train_size` without specifying `test_size`, you'll get a `FutureWarning` letting you know that the behavior is changing in the future (see soon-to-be-merged [scikit-learn/scikit-learn/#7459](https://github.com/scikit-learn/scikit-learn/pull/7459)). I think the vast majority of people are unaware of this difference, and might use `train_size` and `test_size` interchangeably (complementing when switching to the other) --- this might have quietly caused you some issues in the past.
