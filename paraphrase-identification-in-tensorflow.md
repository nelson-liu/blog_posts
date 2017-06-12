I've been loosely hacking on the [Quora Question Pairs](https://www.kaggle.com/c/quora-question-pairs) dataset in my free time to get some more experience working with vanilla Tensorflow for NLP in a practical setting. Yesterday, I opened sourced the code I've written (with some contributions from [Omar Khan](https://github.com/ohkhan), thanks!) and you can [find it on GitHub (nelson-liu/paraphrase-id-tensorflow)](https://github.com/nelson-liu/paraphrase-id-tensorflow).

To be specific, I implemented the following models in the repo:

- A basic Siamese LSTM baseline, loosely based on the model
  in
  [Mueller, Jonas and Aditya Thyagarajan. "Siamese Recurrent Architectures for Learning Sentence Similarity." AAAI (2016).](https://www.semanticscholar.org/paper/Siamese-Recurrent-Architectures-for-Learning-Sente-Mueller-Thyagarajan/6812fb9ef1c2dad497684a9020d8292041a639ff)
  
- A Siamese LSTM model with an added "matching layer", as described
  in
  [Liu, Yang et al. "Learning Natural Language Inference using Bidirectional LSTM model and Inner-Attention." CoRR abs/1605.09090 (2016)](https://www.semanticscholar.org/paper/Learning-Natural-Language-Inference-using-Bidirect-Liu-Sun/f93a0a3e8a3e6001b4482430254595cf737697fa).

- The more-or-less state of the art Bilateral Multi-Perspective Matching model
  from
  [Wang, Zhiguo et al. "Bilateral Multi-Perspective Matching for Natural Language Sentences." CoRR abs/1702.03814 (2017)](https://www.semanticscholar.org/paper/Bilateral-Multi-Perspective-Matching-for-Natural-L-Wang-Hamza/b9d220520a5da7d302107aacfe875b8e2977fdbe).

Anecdotally, when I was first starting out with Tensorflow, by far the most effective learning strategy for me was to (1) read a paper, (2) find an open source implementation of it, and then (3) read through the code. Through this, I reinforced my knowledge of both the work described in paper as well as how people write Tensorflow code in practice.

Unfortunately, the largest issue was mostly (2) --- it's hard to find well-written projects that explain what's going on with minimal magic involved. This is fairly understandable, though, as research code is basically written ... for the sole purpose of research. Other people are rarely going to run your code after the fact, and frankly researchers get a gold star just for open-sourcing anything regardless of quality; there's thus little incentive to actually write your code as if other people would use it.

To this end, I've took painstaking care to try to implement the models "right way" by adhering Tensorflow best practices, document the code (through function docstrings and comments in general), and provide tests (98% coverage + running on a CI server). 

Hopefully someone finds this helpful, and I'm happy to answer questions about the project and/or provide more info about how to get started with Tensorflow.


_Thanks for reading! Feel free to follow me on [GitHub](https://github.com/nelson-liu) or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
