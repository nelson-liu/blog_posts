In my [previous blog post](http://blog.nelsonliu.me/2016/06/22/gsoc-week-4-the-mean-absolute-error-criterion/), I discussed a method for using two heaps to efficiently find the median for use in the MAE criterion for finding the best split. However, the post did not include information on how to extend the method used to efficiently calculate the weighted median. This post will fill those holes, and give a detailed explanation of how I am extending the median calculation method shown in the previous post to deal with the important case of weighted data.

# Recap of the weighted median problem
In my last blog post, I briefly explained the intuition behind the weighted median problem; it's quoted below for your reading convenience:
>A common way to define the median of a set of samples is as the value such that the number of samples above and below the median are both less than or equal to half the total number of samples. The weighted median is thus similar, but we seek to find a value such that the total weights of the samples above and below the median are both less than or equal to half the total weight of all samples. If this seems a bit strange, don't worry! Examples are provided a bit further below.

# Calculating the Weighted Median, given sorted values and weights
Before diving into the intricacies of how to calculate the weighted median of a running stream of numbers, it's important to have an algorithm to calculate the weighted median given a sorted array of values and the associated weights. For example, let's call the array of values `Y` and its associated weights `W`. If we had the data `Y = [4,1,6]` and `W = [3,1,2]`, then we have a sample with a value of 4 and a weight of 3, another sample with a value of 1 and a weight of 1, and a last sample with a value of 6 and a weight of 2. When sorted by value, the data would look like:

```
Y = [1,4,6]
W = [1,3,2]
```

Notice that the `W` array is not sorted, but is rather arranged such that the weight of `Y[i]` is `W[i]`, for all `i` in the python expression `range(len(Y))`. Given this data, to calculate the weighted median, we divide the calculation into two cases.

## Case One: The "Split Median" Case
When calculating the unweighted median, there are cases where it's necessary to take the average of two elements in order to find the median --- I refer to this as the "split median" case, because you need to "split" to find the median. Namely, this is necessary when the number of elements is even. For example, the median of `[1,2,3,4]` is the average of `2` and `3`, or `2.5`. In a more nuanced case, the median of `[3,4,4,5]` is the average of `4` and `4`, which is still `4`.

In the weighted median calculation, there is also a "split median" case. For example, given `Y = [3,4,5,6]` and `W = [1,2,2,1]`, you're essentially calculating the median of `Y = [3,4,4,5,5,6]` with weights `W = [1,1,1,1,1,1]`. This comes out as a "split median", the value `4.5` (the average of `4` and `5`). However, the rule for determining whether a given set of data will need a split median is not the same as in the unweighted case; that is, just because `sum(W) % 2 == 0` (sum of weights is even) does not mean that the median will be a split median. This seems slightly unintuitive, especially considering our previous example of converting the weighted median calculation into an unweighted one simply involved creating a new data array with `len(sum(W))`. 

However, the critical catch is when using non-whole weights. If you're given the data `Y = [1,2]` with weights `W = [1.5,1.5]`, it's quite obvious that this will lead to a split median (`1.5`, the average of `1` and `2`). However, the `sum(W) = 3.0`, which is not an even number. We cannot convert a problem like the above easily to an unweighted median problem, because we cannot express a value occurring with a frequency of `1.5`. We could use tricks like simplifying the weights to `W = [1,1]` (divide by LCM) or `W = [15,15]` (turn decimals into whole numbers), but this gets unwieldy very quickly when dealing with weights that can be irrational floats. 

As a result, we need a different way of determining whether a weighted median calculation will involve taking the average of two values. To do so, it's useful to refer back to the definition of the weighted median problem. We're looking for the value such that the sum of the weights above and below it are equal. Essentially, we want to find whether there is a value that exactly splits the weights in halves. In other words, is there a value of `k` in `range(1, len(W))` such that `sum(W[0:k])` = `sum(W[k:len(W)+1])` = `sum(W) / 2` (Remember that the notation `W[0:k]` are the elements `W[0], W[1], ... W[k-1]`)?

Revisiting the examples above with the definition we just introduced, we can verify that it holds.
Given the data:
```
Y = [3,4,5,6]
W = [1,2,2,1]
```

we can verify that for the value `k = 2`, `sum(W[0:2]) = 1 + 2 = 3` and `sum(W[2:4]) = 2 + 1 = 3`, with additional verification in the fact that `sum(W) / 2 = 3`. You can verify for yourself that valid values of `k` exist that fulfill the property presented above on the datasets with `Y=[1,2]` and `W=[1.5,1.5]`, as well as `Y=[1,2,3]` and `W=[2,1,1]`, indicating that they will indeed involve an average calculation.

Now that we know when a weighted median calculation will involve an average, how do we actually use it to calculate the weighted median? It's actually quite simple -- once we have figured out the value of `k` that indicates that the median will require an averaging operation, the median is simply calculated as \\( \\frac{Y[k-1] + Y[k]}{2} \\). 

Thus, for the example above (`Y = [3,4,5,6]` and `W=[1,2,2,1]`, with `k = 2`), we calculate the median as \\( \\frac{Y[2-1]+Y[2]}{2} = \\frac{Y[1]+Y[2]}{2} = \\frac{4+5}{2} = 4.5 \\).

In the second example above (`Y=[1,2]` and `W=[1.5,1.5]`, with `k=1`), the median is calculated as \\( \\frac{Y[1-1]+Y[1]}{2} = \\frac{Y[0]+Y[1]}{2} = \\frac{1+2}{2} = 1.5 \\). Try calculating the weighted median with these steps for the third example, and verify that the weighted median is equal to `1.5`.

## Case Two: The "Whole" Median Case
When the weighted / non-weighted median calculation does not require taking the average between two values, I refer to the calculation process as calculating a "whole" median. For example, finding the unweighted median of `[1,2,3]` is `2`, a conclusion we can arrive at without taking any averages. In the weighted case, the weighted median given `Y=[3,4,5,6]` with `W=[1,2,1,1]` --- the weighted median is `4`, a calculation that also does not require the averaging operation. Our criteria for when a median calculation is a "Whole Median" case is simple; if the median calculation is not "split", it is whole. In other words, if there is **NO** value of `k` in `range(1, len(W))` such that `sum(W[0:k])` = `sum(W[k:len(W)+1])` = `sum(W) / 2`, the median is a "whole median". If there is no value of `k` in `range(1, len(W))` such that `sum(W[0:k])` = `sum(W[k:len(W)+1])` = `sum(W) / 2`, there must be a value of `k` in the same range such that `sum(W[0:k])` > `sum(W) / 2`. Calculating the value of the median in this "whole" case requires knowing the smallest value of `k` that satisfies the condition above, namely that `sum(W[0:k])` > `sum(W) / 2`.

Looking back at the earlier weighted median example, we can calculate the value of `k`. Given the data `Y=[3,4,5,6]` with `W=[1,2,1,1]`, we can see that:
```
sum(W) / 2 = 2.5
when k = 1, sum(W{0:1]) = W[0] = 1
when k = 2, sum(W[0:2]) = W[0] + W[1] = 1+2 = 3
when k = 3, sum(W[0:3]) = W[0] + W[1] + W[2] = 1+2+1 = 4
...
```
As a result, the correct value of `k` would be `2`, since it is the smallest value such that `sum(W[0:k]) > sum(W) / 2`. With this value of `k` in hand, it's trivial to calculate the median; the median is simply `Y[k-1] = Y[1] = 4`.

## So how does this help us?
Now that we have methods to calculate the weighted median from sorted instances of `Y` and its associated weights `W`, we're quite close to having a working (albeit a bit naive) solution to the weighted median problem. If we modify the min and max heaps to internally be sorted arrays (such that in a min heap, the array is sorted in decreasing order and the array is sorted in increasing order in the max heap), we can calculate the median at any time step in `O(n)` time with the roughly following algorithm (in pseudo-pythonic code):
```
def calculate_weighted_median(med_heap):
    total_weight = sum(med_heap.min_heap) + sum(med_heap.max_heap)
    k,sum = calculate_k(med_heap, total_weight)
    if sum == total_weight: # split median case
        # resize the value of k for accessing elements in each heap
        # return (Y[k-1] + Y[k])/2
    else: # whole median case
        # resize the value of k for accessing elements in each heap
        # return Y[k-1]

def calculate_k(med_heap, total_weight):
    sum_of_w_0_to_w_k_plus_1 = 0
    for k in range(1, len(med_heap.max_heap)):
        # go from left to right in the sorted array, so 
        # start with the max heap and move to the min heap
        sum_of_w_0_to_w_k_plus_1 += med_heap.max_heap[k-1]
        if sum_of_w_0_to_w_k_plus_1 >= total_weight // 2:
            # we have found the value of k
            return (k, sum_of_w_0_to_w_k_plus_1)
    
    # if the for loop above terminates, we move to the min heap
    for k in range(1, len(med_heap.min_heap)):
        sum_of_w_0_to_w_k_plus_1 += med_heap.min_heap[k-1]
        if sum_of_w_0_to_w_k_plus_1 >= total_weight // 2:
            # we have found k, but be sure to add len(med_heap.max_heap)
            # since we restarted iteration from 1
            return (k + len(med_heap.max_heap), sum_of_w_0_to_w_k_plus_1)
```

So in theory, we're done! We can just calculate the median at each time step, and have a functionally correct solution. However, we'd prefer not to calculate the median at each time step because it's fairly expensive, and thus we want to save the values of `k` and `sum_of_w_0_to_w_k_plus_1`, and simply update them each time a new element is added to the set of values and weights we are considering.

## Speeding up calculation by iterative updating
If you start off with an empty list (the first iteration of the running weighted median calculation), and insert an element `2` with weight `2`, you start with `k = 1` and `sum_of_w_0_to_w_k_plus_1 = 2` (NOTE: this is the value of W[0] + W[1] ... W[k], henceforth referred to as simply `sum_w_k` for brevity). This makes sense, because the value of `sum_w_k` is equal to `sum(W[0:k]) = sum(W[0:1]) = W[0]` with `k` being appropriately 1. Using the principles earlier, we can see that `sum_w_k > (total_weight / 2)`, thus indicating that the median is at `Y[k-1], or Y[0]` because it is a "whole median".

At the end of this first time step, you're left with the following state:
```
Y = [2]
W = [2]
k = 1
sum_w_k = 2
total_weight = 2
```

### Second Time Step - inserting below the median
Let's say that in the second time step, you insert the value `1` with an associated weight of `1`. Now, `Y = [1,2]` and `W=[1,2]`. We need to update `k`, `sum\w__k`, and `total_weight` to reflect the new insertion. Updating `total_weight` is trivial, as we just add `1` for a new `total_weight = 3`. Since we inserted something LESS than our original median at `Y[k-1]`, we INCREMENT `k` by one so it now has a value of `2`. Let's make `original_k` the value of our `k` at the first time step, namely `1`. Similarly, we update `sum_w_k` to reflect our new `k`, so now it is equal to `sum(W[0:k]) = sum(W[0:2]) = W[0]+W[1]=3`. 

Given these new values, we have an issue; we cannot the median with these values because we may have broken the condition that `k` must be the SMALLEST number possible that such that `sum(W[0:k]) >= total_weight / 2`. Since we inserted something less than `Y[original_k] = 2`, we know that the median MUST have decreased. As a result, our value of `k` can only `decrease` as well. As as result, we want to see if `k` is still the smallest value such that `sum(W[0:k]) >= total_weight / 2`. To do this, we iteratively try smaller values of `k` in the following fashion:
```
# recall that sum_w_k represents sum(W[0:k])
while(sum_w_k - W[k] >= total_weight / 2 and k != 1):
    k = k - 1
    sum_w_k = sum_w_k - W[k]
```
After this while loop completes, it is guaranteed that `k` is the smallest value in `range(len(Y))` such that `sum(W[0:k+1]) >= total_weight / 2`, and that `sum_w_k` is updated accordingly. As a result, after the second time step, our state looks like:

```
Y = [1,2]
W = [1,2]
k = 2
sum_w_k = 3
total_weight = 3
```

If we wanted to find the median at this time step, we would first determine whether it is a split or whole median by checking if `sum_w_k == total_weight / 2`; since it does not, we know that it is a whole median, and the median has the value `Y[k-1] = Y[1] = 2`. In this second time step, we covered the case of inserting a value less than the current median.

### Third and Fourth Time Step - inserting above the median
In the third time step, we check the case of inserting a value greater than the current median, say the value `5` with a weight of `3`. Now, `Y = [1,2,5]` and `W = [1,2,3]`. Since we inserted something above the median, we do not change the value of `k` and `sum_w_k`. However, we face the opposite problem as when inserting below the median. When inserting below the median, you lose the guarantee that `k` must be the SMALLEST number possible that such that `sum(W[0:k]) >= total_weight / 2`. On the other hand, when inserting above the median, you lose the guarantee that `sum(W[0:k]) >= total_weight / 2` in the first place! Our new total weight is `6`. we can see that `sum(W[0:k]) = sum(W[0:2]) = 3 >= total_weight / 2` still holds true, so we do not need to update anything further. However, when calculating the median, since `sum_w_k == total_weight / 2`, it must be a split median. As a result, we apply the formula that \\( \\frac{Y[k-1] + Y[k]}{2} = \\frac{Y[2-1] + Y[2]}{2} = \\frac{Y[1] + Y[2]}{2} = \\frac{2 + 5}{2} = 3.5 \\) to calculate the correct split weighted median.

After the 3rd time step, the state is:
```
Y = [1,2,5]
W = [1,2,3]
k = 2
sum_w_k = 3
total_weight = 6
```

In the fourth time step, I'll demonstrate the case of inserting a value greater than the current median and having to update `k`. Let's say that we insert the value `8` with a weight of `5`. As a result, our `Y = [1,2,5,8]` and our `W = [1,2,3,5]`. Once again, since we inserted above the median, we initially do not have to change the value of `k` and `sum_w_k`. We need to check whether our current value of `k = 2` is the smallest number possible that fulfills `sum(W[0:k]) >= total_weight / 2`. `total_weight` is now equal to `11`, since we add `5`. Because we insert something above the median, the median can only increase; thus, we look for higher values of `k`. Note that `sum(W[0:2]) = 3`, which IS NOT greater than `total_weight / 2= 11/2 = 5.5`.  To look for the new value of `k`, we iteratively try higher and higher values until the condition that `sum(W[0:k]) >= total_weight / 2` is met. To illustrate:
```
iteration 1: Try k = 3 (2+1)
set sum_w_k = sum(W[0:3]) = W[0]+W[1]+W[2] = 6
is sum_w_k >= total_weight / 2? yes, since 6 > 5.5
We have found the lowest value of k fulfilling our target condition, and thus we break
```
As a result, the new value of `k = 3`. With this, we can calculate the new median after inserting the new value. since `sum(W[0:k]) > total_weight / 2`, we know the median is whole. Thus, the median is simply `Y[k-1] = Y[3-1] = Y[2] = 5`. This is the correct answer, given the following state after step 4:
```
Y = [1,2,5,8]
W = [1,2,3,5]
k = 3
sum_w_k = 6
total_weight = 11
```

### Step 5 - Inserting at the median
Obviously, inserting the values at the median will not change the median. However, it's important to ensure that we properly modify the other variables in order to facility correct future insertions. Given that the median in the previous state is `5`, we will insert the value `5` with the weight `1`. As a result our `Y = [1,2,5,8]` (unchanged) and `W = [1,2,4,5]`. We accordingly update `total_weight = 12`. Now, we have to update the value of `k` and `sum_w_k`. The value of `k` should not change, since the median itself does not change. However, we do need to add the weight that we just inserted (`1`, in this case) to `sum_w_k`, making it `7`. Calculating the median is trivial; we see that `sum_w_k` > `total_weight / 2`, so the median is `Y[k-1]`, where `k` remained unchanged at `3`. Thus, the median is still `Y[2] = 5`. The state after step 5 is:
```
Y = [1,2,5,8]
W = [1,2,4,5]
k = 3
sum_w_k = 7
total_weight = 12
```

### Step 6 - Post-Median Insertion Sanity Check
As a last step, we will insert another value just to ensure that we performed the proper actions in step 5. In this case, we'll choose to insert an extreme value below the median --- we will insert the value `0` with a weight of `14`. Now, `Y = [0,1,2,5,8]` and `W = [14,1,2,4,5]`. We update `total_weight = 26` and `sum\w__k = 21`, and `k=4`. Since we inserted below the median, we need to search `k` values that are closer to 0. Our search is detailed below, starting from our base value of `k=4`:
```
# iteration 1, verify that current k = 4 doesn't work
is sum_w_k >= total_weight / 2? yes, since 21 > 13.

# iteration 2, check if k = 3 also fulfills the condition
set sum_w_k = sum(W[0:3]) = W[0]+W[1]+W[2] = 14+1+2 = 17.
is sum_w_k >= total_weight / 2? yes, since 17 > 13

# iteration 3, check if k = 2 also fulfills the condition
set sum_w_k = sum(W[0:2]) = W[0]+W[1] = 14+1 = 15.
is sum_w_k >= total_weight / 2? yes, since 15 > 13

# iteration 4, check if k = 1 also fulfills the condition
set sum_w_k = sum(W[0:2]) = W[0] = 14.
is sum_w_k >= total_weight / 2? yes, since 14 > 13

# since k = 1 is the minimum value, we break.
```

With our value of `k=1`, we can calculate the new weighted median. we see that `sum_w_k > total_weight / 2`, thus this is a "whole median", and we use `Y[k-1] = Y[0]` to get a weighted median result of `0`, which is correct. After step 6, the program state is:
```
Y = [0,1,2,5,8]
W = [14,1,2,4,5]
k = 1
sum_w_k = 14
total_weight = 26
```

We can insert more arbitrary numbers and weights, and this system will be able to handle it.

## Conclusion
In this blog post, I talked about an empirical method I am using to quickly calculate the weighted median given a running stream of integers. This will be used as a special case of the `MedianHeap` implementation already in place to handle unweighted values. I'm struggling to find formal mathematical proof that this works, and would highly appreciate if you could comment below if you have any thoughts especially regarding its correctness. It seems intuitive to me and appears to work for a wide variety of cases, but that doesn't mean it's correct in all cases.

If you have any questions, comments, or suggestions, you're welcome to leave a comment below.

Thanks to my mentors [Raghav RV](https://github.com/raghavrv) and [Jacob Schreiber](https://github.com/jmschrei) for their input on this problem; we've run through several solutions together, and they are always quick to point out errors and suggest improvements.

_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu)  if you want to track the progress of my Summer of Code project, or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
