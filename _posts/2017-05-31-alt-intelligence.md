---
layout:     post
title:      "Alternative Intelligence"
date:       2017-05-31 14:57:00
summary:    "Donald Trump simulator using RNNs"
categories: things stuff
---

This is a technical overview of my Donald Trump Twitter simulator, which can be found at [https://kevin-albert.github.io/bingbingbongbongbingbingbing](https://kevin-albert.github.io/bingbingbongbongbingbingbing).  

## Introduction

An ongoing topic of interest in politics and news, Donald Trump's twitter account is also an ever-growing source of data for natural language analysis. By applying a widely-used character-level deep RNN model ([Generating Sequences With Recurrent Neural Networks. Graves, 2014](https://arxiv.org/pdf/1308.0850.pdf), [The Unreasonable Effectiveness of Recurrent Neural Networks. Karpathy, 2015](https://karpathy.github.io/2015/05/21/rnn-effectiveness/)), we are able to build a Donald Trump simulation that is parameterized in real time with an adjustable softmax temperature.  

## Training Data

The training data is simply the corpus of Donald Trump's tweets from 2009 to 2017, including him self-promoting his book, supporting Mitt Romney's presidential campaign, criticizing Barack Obama and Rosie O'Donnell, measuring Trump Tower, and of course telling us how powerful his twitter had become - approximately 3.3MB of raw text, generously provided by the [Trump Twitter Archive](http://trumptwitterarchive.com/). The data is all concatenated into a single file, with each tweet delimited by the symbol `^` (which signals the network that a new sequence is starting). During training, the entire file is scanned, and the internal state of the recurrent layers is reset every 10000 characters.

## Architecture:

* **Layers**: 166 (Input) -> 256 (LSTM) -> 256 (LSTM) -> 256 (LSTM) -> 166 (Output)
* **Optimization**: Nesterov Accelerated Momentum
* **BPTT Sequence Length**: 100
* **Learning Rate**: 0.0005
* **Learning Rate Decay Factor**: 0.975
* **Regularization**: Add gaussian noise to output with standard deviation 0.01  

## Training

Training is performed in sub-sequences of length 100, using truncated backpropagation through time and Nesterov momentum (a simple modification to normal SGD needing $$\Theta(n)$$ additional storage). For each forward pass, the network is presented with a single input character. The input signal is propagated forward through the network, and a probability distribution is produced, attempting to predict what the next character will be. Because the next character in the training sequence is known, its target probability is 1 while the target probabilities for every other character for that step are 0. In this manner, the model iterates forward through 100 characters and then backward through those 100 characters, accumulating error signals. After the forward / backward loops, every network parameter $$w$$ has some $$\frac{\partial E}{\partial w}$$ for the subsequence. Nesterov momentum is used to adjust $$w$$ accordingly.  

## Sampling 

After several epochs, the network can be sampled to synthesize text snippets similar to Donald Trump tweets. The sample process is simple: First, present the network with the start character `^` and run the forward pass. The network outputs a probability distribution, from which the next character can be sampled (In many cases, `^` is succeeded by `T`). The sampled character is pushed into the output and then fed back into the network as the input to the next forward pass. This is repeated until the next `^` is encountered, or a hard limit (say, 140 characters) is reached.  

As I previously mentioned, softmax temperature can be adjusted to tune the network output. To see how this works, remember that softmax for a single output value $$y_i$$ is calculated as:  
\begin{align}
p_i &= \frac{e^{y_i}}{\sum_{j = 1}^n e^{y_j}}
\end{align}

Essentially, softmax first exponentiates everything and then normalizes so that the resulting vector sums to 1, creating a probability distribution. The adjust the output variance, a temperature factor $$t$$ can be added in during sampling, so that $$y_i$$ is first divided by $$t$$. Since $$\frac{1}{t}$$ applies a linear scale to outputs and $$e^{y_i}$$ applies an exponential scale, the change in difference between values as $$\frac{1}{t}$$ grows enhances the differences when exponentiating, causing the dominant probabilities to increase and the unlikely ones to further decrease. In this application, sampling with $$t$$ close to 0.5 tends to produce realistic output, while lower values tend toward generating cycles such as "I will be a great the people of the people of the people the people of the..." and higher values start to produce strings of random characters. The web page randomly assigns $$t$$ to a value between 0.2 and 1.1 for each tweet it generates.

## Implementation

I wrote this network in C++ using [Eigen](http://eigen.tuxfamily.org/), to take advantage of CPU-optimized linear algebra routines (the code is very specific to my machine, which doesn't have a CUDA-enabled graphics card or anything). It's got OMP, Intel MKL, and SSE2 turned on. Pretty sure most  (if any) performance improvement comes from those last two, but I haven't taken the time to do any profiling. My source code can be found on [Github](https://github.com/kevin-albert/lstm-cpp).  

Included in the C++ app is a tool to export the network parameters to JavaScript. I then rewrote the forward pass using [https://github.com/hiddentao/linear-algebra](https://github.com/hiddentao/linear-algebra), stuck it in a Web Worker, and dressed it up a little bit. The page repeatedly calls the web worker which performs the sampling routine until a `^` is reached or 140 characters have been generated. The finished product can be experienced at [https://kevin-albert.github.io/bingbingbongbongbingbingbing](https://kevin-albert.github.io/bingbingbongbongbingbingbing).  

## Conclusion

The resulting content is nearly indistinguishable from the real twitter account. Open areas of study involve assigning higher probability to tweets which illicit more positive feedback (number of likes / retweets), and/or creating an AI-driven twitter account with the purpose of convincing follower that it is the real Donald Trump while in fact the one in the White House is an imposter holding the real one in captivity but somehow the real one gained access to a mobile device and created a Twitter account and it slowly gains the trust of all top government officials and assumes the role of President and then we have a computer in charge of a global superpower.  

