# Overview
A word embedding is a paramaterized function which maps words in some language to high-dimensional vectors. Converting words to such vector embeddings before passing them into deep neural networks has proved to be a highly effective technique for text classification tasks. 

Presented here is a novel method to modify the embeddings of a word in a sentence with its surrounding context using a biderectional Recurrent Neural Network (RNN). The hypothesis being tested is that these modified embeddings are a better input for text classification networks. This repository contains code implementations, experimental results and visualizations.

# Naive Example
Given the embeddings for all the words in a sentence like *'the quick brown fox jumps over the lazy dog'*, the proposed model modifies the existing embedding for *'fox'* to incorporate information about it being *'quick'* and *'brown'*, and the fact that it *'jumps over'* the *'dog'* (whose updated embedding now reflect that it is *'lazy'* and got *'jumped over'* by the *'fox'*). 

Applied in combination with pre-trained word embeddings (like [word2vec](https://code.google.com/archive/p/word2vec/) or [GloVe](http://nlp.stanford.edu/projects/glove/)) which encode global syntactic and semantic information about words such as *'fox'* and *'dog'*, the method adds local context to these embeddings based on surrounding words. The new embeddings can then be fed into a text classification network.

# Model
![Bidirectional RNN layer](res/bidirectional-rnn.png?raw=true)

Given the word embeddings for each word in a sentence/sequence of words, the sequence can be represented as a 2-D tensor of shape (`seq_len`, `embedding_dim`). The following steps can be performed to add infomation about the surrounding words to each embedding- 

1. Pass the embedding of each word sequentially into a forward-directional RNN (fRNN). For each sequential timestep, we obtain the hidden state of the fRNN, a tensor of shape (`hidden_size`). The hidden state encodes information about the current word and all the words previously encountered in the sequence. Our final output from the fRNN is a 2-D tensor of shape (`seq_len`, `hidden_size`). 

2. Pass the embedding of each word sequentially (after reversing the sequence of words) into a backward-directional RNN (bRNN). For each sequential timestep, we again obtain the hidden state of the bRNN, a tensor of shape (`hidden_size`). The hidden state encodes information about the current word and all the words previously encountered in the sequence. Our output is a 2-D tensor of shape (`seq_len`, `hidden_size`). This output is reversed again to obtain the final output of the bRNN. 

3. Concatenate the fRNN and bRNN outputs element-wise for each of the `seq_len` timesteps in the two outputs. The final output is another 2-D tensor of shape (`seq_len`, `hidden_size`).

**The fRNN and bRNN together form a bidirectional RNN. The difference between the final outputs of fRNN and bRNN is that at each timestep they are encoding information about two different sub-sequences (which are formed by splitting the sequence at the word at that timestep). Concatenating these outputs at each timestep results in a tensor encoding information about the word at that timestep and all the words in the sequence to its left and right.**

The cells used in the RNNs are the [Long Short-term Memory (LSTM)](http://deeplearning.cs.cmu.edu/pdfs/Hochreiter97_lstm.pdf) cells, which are better at capturing long-term dependencies than vanilla RNN cells. This ensures our model doesn't just consider the nearest neighbours while modifying a word's embedding.

# Implementation
The code implements the proposed model as a pre-processing layer before feeding it into a [Convolutional Neural Network for Sentence Classification](https://arxiv.org/abs/1408.5882) (Kim, 2014). Two implementations are provided to run experiments- one with [tensorflow](https://www.tensorflow.org/) and one with [tflearn](http://tflearn.org/) (A high-level API for tensorflow). Training happens end-to-end in a supervised manner- the RNN layer is simply inserted as part of the existing model's architecture for text classification.

The tensorflow version is built on top of [Denny Britz's implementation of Kim's CNN](https://github.com/dennybritz/cnn-text-classification-tf), and also allows loading pre-trained word2vec embeddings. 

Although both versions work exactly as intended, the repository currently contains results from experiments with the tflearn version only. More results will be added as soon as training converges!

# Datasets
The dataset chosen for training and testing the tensorflow code is the [Pang & Lee Movie Reviews](http://www.cs.cornell.edu/people/pabo/movie-review-data/) dataset. For the tflearn version, we experiment on the [IMDb Movie Reviews Dataset](http://www.iro.umontreal.ca/~lisa/deep/data/imdb.pkl) by UMontreal. Classification involves detecting positive or negative reviews in both cases.

# Experiments
The following three models were considered- (Implementations can be found in `/tflearn`)

1. Kim's [baseline CNN model](../master/res/cnn-128.png?raw=true) without the RNN layer, `embedding_dim = 128`, `num_filters = 128` **[ORANGE]**
2. [The proposed model](../master/res/lstm%2Bcnn-128.png?raw=true), `embedding_dim = 128`, `rnn_hidden_size = 128`, `num_filters = 128` **[PURPLE]**
3. [The proposed model with more capacity](../master/res/lstm%2Bcnn-300.png?raw=true), `embedding_dim = 300`, `rnn_hidden_size = 300`, `num_filters = 150` **[BLUE]**

All models were trained with the following hyperparameters using the Adam optimizer- `num_epochs = 100`, `batch_size = 32`, `learning_rate = 0.001`. Ten percent of the data was held out for validation.

# Results
Training Accuracy-
![Training Accuracy](res/acc.png?raw=true)

Training Loss- 
![Training Loss](res/loss.png?raw=true)

It is clear that training converges for all three models.

Validation Accuracy-
![Validation Accuracy](res/acc-val.png?raw=true)

Vallidation Loss-
![Validation Loss](res/loss-val.png?raw=true)

Higher Validation Accuracy (~3%) and lower Validation Loss for the model compared to the baseline suggests that adding the bidirectional RNN layer after the word embedding layer improve a generic text classification model's performance. More rigourous experimentation needs to be done to confirm this hyposthesis.

**An unanswered question is whether the bump in accuracy is because the RNN layer actually adds contextual information to independent word embeddings or simply because of more matrix multiplications by the network. However, adding more hidden units to the RNN layer does not lead to drastic changes in accuracy, suggesting that the former is true.**

It is also extremely worrying to see the validation loss increasing instead of decreasing as training continues. This issue needs investigation.

# Ideas and Next Steps
1. Visualizations of the modified embeddings in a sequence can be compared to their original embeddings to confirm that their modification is due to their surrounding words and is not random.

2. An `n` layer vanilla neural network for text classification can be compared to a model with the RNN layer followed by an `n-1` layer vanilla network. This should be a 'fairer fight' than a deep CNN vs RNN followed by deep CNN.

3. Experiments can be carried out on initialization using pre-trained embeddings passing them as trainable vs non-trainable parameters to the RNN layer; i.e. whether or not we backpropagate errors into the original embedding layer.

4. Experiments can be performed to determine the optimum depth of the RNN layer for different kinds of models on top of it. (Currently it is a single layer, but the concept can easily be extended to multilayer bidirectional RNNs.)

5. Cross validation should be performed to present results instead of randomly splitting the dataset.

# Usage
Tensorflow code is divided into `model.py` which abstracts the model as a class, and `train.py` which is used to train the model. It can be executed by running the `train.py` script (with optional flags to set hyperparameters)-
```
$ python train.py [--flag=1]
```

Tflearn code can be found in the `/tflearn` folder and can be run directly to start training (with optional flags to set hyperparameters)-
```
$ python tflearn/model.py [--flag=1]
```

The summaries generated during training (saved in `/runs` by default) can be used to visualize results using tensorboard with the following command-
```
$ tensorboard --logdir=<path_to_summary>
```
