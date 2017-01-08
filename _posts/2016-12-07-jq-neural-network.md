---
layout:     post
title:      "Machine Learning With Jq"
date:       2016-12-07 12:49
summary:    "A basic neural network library written in jq"
categories: things stuff
---

Using the JSON-processing language [jq](https://stedolan.github.io/jq/ "jq"), I built a neural network that classified digits in the [MNIST handwritten digit database](http://yann.lecun.com/exdb/mnist/ "MNIST handwritten digit database") with 94.07% accuracy (only 1 training epoch). While this pales in comparison to other networks that have classified the same dataset almost perfectly, it does demonstrate the expressive power of a tool that is well-known but typically only used for pretty-printing JSON.  

Because data in jq is immutable, the state of the network (weights, biases, activations, etc) is accumulated through a tail-recursive reduction across the input dataset. The entire neural network library is only 160 lines long (including a decent amount of comments). Running on a t2.micro, the computation (one pass over the test dataset and one over the training dataset) took about a week.  

Reference info about the network itself:
* **Layers**: 784 (input) -> 100 (sigmoid) -> 10 (sigmoid)
* **Cost function**: Cross-entropy
* **Training method**: Plain old SGD
* **Learning Rate**: 0.1
* **Epochs**: 1

No batching, regularization, or anything clever.  

Once the looping method was established (jq's `reduce` function), it was a simple matter of building up some utility functions for dot products, matrix multiplications, sigmoid and derivative-of-sigmoid<sup>-1</sup> and then wiring together the logic for the forward and backward passes. After that the biggest challenge was getting the data into JSON.  

You can check out the codebase: [https://github.com/kevin-albert/jq-neural-network](https://github.com/kevin-albert/jq-neural-network "Github"). And for a quick-n-dirty mathematical reference check out [blog post](http://briandolhansky.com/blog/2014/10/30/artificial-neural-networks-matrix-form-part-5 "Artificial Neural Networks: Matrix Form (Part 5)") by Brian Dolhansky.

