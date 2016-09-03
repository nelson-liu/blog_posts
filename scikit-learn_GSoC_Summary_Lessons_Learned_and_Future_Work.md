This summer, I was quite fortunate to work on the [scikit-learn](https://github.com/scikit-learn/scikit-learn) project with my mentors [Jacob Schreiber](https://github.com/jmschrei) and [Raghav RV](https://github.com/raghavrv) as part of the [Google Summer of Code Program](https://developers.google.com/open-source/gsoc). I worked on various features for the [`tree` module](http://scikit-learn.org/stable/modules/classes.html#module-sklearn.tree), and I'd like to take a moment now that the program is over to summarize what I've done, recall the many things learned over a plethora of successes and failures, and talk about future work to be done in the space.

# Listing of Work Completed

So, what did I even do during the summer? A summary listing of my contributions is below.

- Implemented mean absolute error (MAE) tree node impurity criterion for use in [`DecisionTreeRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.tree.DecisionTreeRegressor.html), [`RandomForestRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.RandomForestRegressor.html), [`ExtraTreesRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.ExtraTreesRegressor.html), [`GradientBoostingRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.GradientBoostingRegressor.html), and [`GradientBoostingClassifier`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.GradientBoostingClassifier.html) ([PR #6667](https://github.com/scikit-learn/scikit-learn/pull/6667)).

- Implemented impurity-based early stopping for decision tree growth, used through a construction parameter `min_impurity_split` in the [`DecisionTreeRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.tree.DecisionTreeRegressor.html), [`DecisionTreeClassifier`](http://scikit-learn.org/dev/modules/generated/sklearn.tree.DecisionTreeClassifier.html), [`RandomForestRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.RandomForestRegressor.html), [`RandomForestClassifier`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.RandomForestClassifier.html), [`RandomTreesEmbedding`](scikit-learn.org/dev/modules/generated/sklearn.ensemble.RandomTreesEmbedding.html), [`ExtraTreesRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.ExtraTreesRegressor.html), [`ExtraTreesClassifier`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.ExtraTreesClassifier.html), [`GradientBoostingRegressor`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.GradientBoostingRegressor.html), and [`GradientBoostingClassifier`](http://scikit-learn.org/dev/modules/generated/sklearn.ensemble.GradientBoostingClassifier.html) classes ([PR #6954](https://github.com/scikit-learn/scikit-learn/pull/6954)).

Throughout the summer, I...

- committed 999 lines of code
- wrote 7 Google Summer of Code-related blog posts
- helped new contributors successfully make their first contributions at the Scipy 2016 scikit-learn sprint
- participated in reviews of 67 contributed pull requests, for example:
  - [ (#7164) fixing ElasticNet sparse decision function to match output with dense](https://github.com/scikit-learn/scikit-learn/pull/7164)
  - [ (#7049) load\_iris dataset: added return\_X\_y option](https://github.com/scikit-learn/scikit-learn/pull/7049)
  - [ (#6966) make confusion\_matrix\_plot.py example more copy-pastable](https://github.com/scikit-learn/scikit-learn/pull/6966)
- had 14 (including the major two mentioned above) of my pull requests in total merged

all between April 22nd to August 10th, when this post was first written.

# Technical Skills Developed
## Cython
Since much of the `tree` module is written in Cython, it was naturally important for me to acquire a good understanding of the language, how it works, and its various idiosyncrasies. I had never used any C-flavored languages before Cython, so it took a bit of time to get used to the slightly different style of development and concepts like manual handling of pointers, freeing memory, the lack of `IndexError`s when dealing with arrays of pointers, etc. Looking back, I'm happy that I got the chance to work with the language, and I think it'll prove useful in the future for both future contributions and for my own projects and research.

## Decision Trees
To effectively contribute to the `tree` module, I had to know the ins and outs of decision trees. This includes both theoretical underpinnings, as well as scikit-learn's own personal implementation. There were a few sources that were quite helpful for developing this knowledge, mainly Leo Breiman's _Classification and Regression Trees_ [1] for the theoretical side and Chapter 5 Gilles Louppe's PhD thesis _Understanding Random Forests_ [2] for learning more about scikit-learn's implementation.

# Lessons Learned
While I learned about useful tools like Cython and machine learning in general, I also gained some more general advice that I will be sure to carry on into the future.

## Never assume something is easy
When writing my proposal, my mentors and I assumed that implementing MAE would be a straightforward affair. Indeed, writing the first version was quite simple. However, making it efficient required much more. As a result, we spent far more time on the MAE criterion than initially planned, which limited the amount of time I had to work on other things originally outlined in the proposal.
Thus, it's important to be flexible and be willing to change your work priorities when obstacles come up.

## Always ask for help if you need it
I was lucky to have easy access to my mentors through a Google Hangouts conversation, and thus we frequently talked about the various algorithmic tricks we were thinking about for MAE and other aspects of the code. Additionally, we had several video calls to discuss anything that was hard to convey by internet message. My progress was catalyzed immensely by their constant availability, and I'm extremely happy that I asked for help whenever I needed it.

# Future Work to be Done
There is a plethora of future work to be done on the `tree` module, which has been distended to and has become a bit overgrown. There's a high start-up cost to initially contributing to this module, as it's quite complex (seeing as the code is spread across multiple files) and a substantial amount is written in Cython, which takes some getting used to. There are two main goals for contributions to make in the future (which I've begun to work on already):

- Optimize MAE for large datasets, as it's still quite slow
- Add post-pruning to decision trees

Participating in Google Summer of Code has brought me closer to the group of amazing developers and people that is the scikit-learn community. I look forward to working on the issues above and contributing to scikit-learn with code and reviews for years to come, and I couldn't be more satisfied with the experience.

References
\----------------

[1] Breiman, Leo. Classification and Regression Trees. Belmont, CA: Wadsworth International Group, 1984. Print.

[2] Louppe, Gilles. "Understanding random forests: From theory to practice." arXiv preprint arXiv:1407.7502 (2014).
