In the fall, I was awarded a Mary Gates Research Scholarship for a project seeking to extract phonesthemes (submorphemic sound clusters that seem to have meaning, more on what this actually means below), advised by Noah Smith and Gina-Anne Levow. The award stipulates that I present my work at the Undergraduate Research Symposium at UW, so I recently did so and figured it would be an apt time to write (a somewhat approachable, I hope) post about what it's all about.

First things first, here's an image of the poster (click to see a larger PDF): 
[![Image of the poster, click to see PDF.](/content/images/2017/05/nfliu_symposium_phonesthemes_poster_img.png)](http://nelsonliu.me/files/nfliu_symposium_phonesthemes_poster.pdf)

# Background / Summary

This work mainly concerns the idea that language is arbitrary, which is a deep seated notion in linguistics (especially as one of Hockett's design principles of human languages that separate them from animal communication). Briefly, this is the idea that the mapping between the form of a word (in letters or sounds) and the object that it represents is random. For example, there's nothing about the physical shape or appearance of an apple that suggests that the proper name for it is "apple". A notable exception to the rule is onomatopoeia (words that are what they sound like, e.g. "bang", "boom"), but we'll leave that aside for now.

In past psycholinguistics work, people have proposed the existence of "phonesthemes", submorphemic clusters of sounds that carry meaning. To unpack that statement a little bit, I'll start with a brief tangent into what morphemes are.

Morphemes are defined as "a meaningful morphological unit of a language that cannot be further divided", i.e. it is the smallest unit of meaning. Submorphemic simply means that they exist below the morpheme level. For example, the word-initial sound cluster `gl` is a often-cited phonestheme since it occurs in many words that have to do with light or vision (e.g. `glow`, `glint`, `glamor`, `glisten`, `gloss`, `glaze`, etc.) [1] Thus, the question is: how can we extract these phonesthemes from a language?

To do so, we essentially employed sparse regression with the Elastic Net [2] to select the subset of phoneme features that are most predictive of the final word vector, with the intuition that sound clusters that are predictive of the word vector (and thus the meaning of the word) would thus be phonesthemic.

# How I Became Interested In The Problem

This work was largely inspired by work that I saw at NAACL 2016 [3] which sought to quantify the reality of phonesthemes in English. In [the paper (Abramova et. al, 2016)](http://aclweb.org/anthology/N/N16/N16-1038.pdf), they employed a variety of methods to 1) validate phonesthemes previously proposed in psycholinguistics literature and 2) try to automatically get glosses (meanings) for each of the phonesthemes.

I thought the work was super interesting, since I had learned about the arbitrariness of language in my introductory linguistics courses and this seemed to provide a relatively convincing, data driven argument to put that into question. I was really fascinated by the prospect of trying to extract phonesthemes from a language, and this is what spurred my work in this project.

# Method

To extract phonesthemes from a lexicon, we need representations of each words' meaning. Thus, we turned towards word vectors. I won't go into much detail about how word vectors work or how they're trained, but the basic intuition is that they encode the meaning of a term in a vector. They've got some amazing properties; [this blog post (Deep Learning, NLP, and Representations by Chris Olah)](http://colah.github.io/posts/2014-07-NLP-RNNs-Representations/) is a great place to read more about them

## Data

Our starting collection of word vectors came pretrained with GloVe from the Stanford NLP Group; we aggressively filtered this set in an attempt to ensure that we had only English words, and that they were not spurious in meaning (e.g. "the", "that", etc). We used a frequency-based filter based on word counts in the Gigaword Corpus, and removed all the words that showed up less than 10,000 times (too rare) as well as those that showed up in more than 50% of the documents (too common). We also filtered words that had the same lemma (e.g. "dogs", "doggie", "dog" shouldn't all be in the corpus, we only want "dog" in order to avoid overweighting words that simply show up in more forms).

Furthermore, because phonesthemes are an inherently phonetic phenomenon (and spelling does not always equate pronounciation, e.g. "phone" and "foam"), we used CMUDict to convert the filtered words into phonemes (throwing out the words that didn't show up in CMUDict). Originally, we were using a seq2seq G2P model here to try to convert words to phonemes, but the word error rate was unacceptably high and caused too much noise.

With this, we had the basic ingredients we needed for our model. Our input is the phoneme representations of the words, and we seek to predict the word vector.

## The Model

The model had two "stages" -- the first was a normal, unregularized linear regression model to predict word vectors from morpheme features, and the second was an Elastic Net model (think L1 and L2 regularization in one model) to do sparse feature selection (courtesy of the L1 component).

### Model Stage 1: Morpheme Unregularized Linear Regression

In the first stage, we first collected a list of bound morphemes from the CELEX2 database (think suffixes like `-ation`, `ly` and prefixes like `un`, `non`, `re`). We then featurized each word by its constituent morphemes. For example, say we a set of morphemes \\( S \\), with \\( |S| \\) elements. if the word in question is `nonconfigurational`, it'd be broken up into `non-configure-ation-al`. Note that `non`, `ation`, and `al` are all morphemes, so we would have a vector zeroes of length \\( |S| \\) , with 3 1's in the indices that represent the morpheme features for `non`, `ation`, and `al`.

Let's call this morpheme-featurized collection of words \\( X\_{morph} \\). We want to fit a model of the form:
$$ y = WX\_{morph} $$

which is just simple linear regression. After doing so, we can extract the residuals of \\( X\_{morph} \\) by getting the predicted vectors for \\( X\_{morph} \\) (let's call it \\( \\hat{y} \\)) and calculating \\( residuals = \\hat{y} - y \\).

The intuition behind this first step is that we do not want the morphemes to show up in the feature selection (and they will, since they are _very_ strong indicators of a word vector) since we are interested in phonesthemes. Taking the residuals of the new model as our target basically has the effect of us "subtracting out" the morpheme meanings so they have a lesser effect on our phonestheme extraction.

### Model Stage 2: Phonestheme Extraction by Feature Selection with Elastic Net Feature Selection

In this second part of the model, we use the Elastic Net to extract phonesthemes. We featurize each word by the word-initial phoneme bigram. Bigram indicates that we're taking a cluster of length 2. For example, we can represent the word "hello" phonemically as `"HH EH L OW"`. The word-initial phoneme bigram is just the first two phonemes, which would be `"HH EH"` in this case. 

Let's call this morpheme-featurized collection of words \\( X\_{phone} \\). We fit an Elastic Net model, which finds the \\( W \\) minimizing:

$$ J(W) = ||residuals - X\_{phone}W||^2 + \\lambda\_2 ||W||^2 + \\lambda\_1 ||W||_1$$

If you're familiar with regression methods, you'll notice that this is very similar to the Lasso and the Ridge (in fact, it has the penalty terms of both)! The Elastic Net improves on the Lasso by resolving the "grouped selection problem"; if there is a group of highly correlated variables, the Lasso tends to select one variable from a group and ignore the others. The Elastic Net overcomes this by adding the quadratic penalty \\( ||W||^2 \\) term (which alone is ridge).

Note that in the above equation, \\( \\lambda\_1 \\) and \\( \\lambda\_2 \\) are hyperparameters that control the weight of each loss term (L1 and L2, respectively). These were set via cross-validation (with regards to minimizing the MSE for Elastic Net model predicted vs residual vectors).

We rank the features by importance, with the idea that important features are more likely phonesthemes.

# Results

Surprisingly, of the top 10 features selected by the model (see the poster for the table), 8 of them are previously proposed phonesthemes and 1 is a convincing one that has not been yet proposed. There was still one morpheme left in the results, but I was nonetheless impressed by how effective the morpheme regression was (without it, things like `un`, `non`, etc. would show up as well). This seems to validate at least that it somewhat works, so it'd be exciting to what this method comes up with for other languages.

# Future Work

I think that it'd be interesting to see if these phonemes can be used to get better representations for words. For example, in machine translation, QA, and reading comprehension, people have been using deep neural nets run over randomly-initialized character vectors to get better-than-random representations for unknown words. I wonder whether there's utility in using phoneme representations, since it might be believable that there are universal phoneme representations across languages. 

I hope this was an interesting read, and I'd love to hear your thoughts in the comments / via email! Feel free to let me know if you have questions as well.

_Thanks for reading! Feel free to follow me on [GitHub](https://github.com/nelson-liu) or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._

## References

[1] Hutchins, Sharon Suzanne. 1998. The psychological reality, variability, and compositionality of English phonesthemes. Atlanta: Emory University dissertation.

[2] Zou, H. and Trevor, T. (2005). Regularization and Variable Selection via the Elastic Net. *Journal of the Royal Statistical Society*, Series B, 67(2), 301-320

[3] Abramova, Ekaterina and Raquel Fernández. “Questioning Arbitrariness in Language: a Data-Driven Study of Conventional Iconicity.” HLT-NAACL (2016).
