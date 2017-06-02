---
layout:     post
title:      "The Pumping Lemma Explained"
date:       2017-05-04 15:31:00
summary:    "Why is this so confusing?"
categories: things stuff
---

If you are or have ever been an Undergraduate Computer Science student (as I was when I first saw this) then you've probably heard of the Pumping Lemma. It comes in two main flavors &ndash; the [pumping lemma for Regular Languages](https://en.wikipedia.org/wiki/Pumping_lemma_for_regular_languages) and the [pumping lemma for context-free languages](https://en.wikipedia.org/wiki/Pumping_lemma_for_context-free_languages). Every explanation of the pumping lemma I've read is (in my opinion) unnecessarily formal, so I'm going to try and share my understanding of the intuition behind it:  

## For finite languages
There is no string that can be pumped. This is confusing because it seems to contradict the pumping lemma. Actually, the lemma remains true by a technicality: Set the pumping length $$p$$ to be longer than the longest string in your finite language. The pumping lemma is now *vacuously* true because there are 0 strings against which it can be tested. Now the lemma is technically correct - the best kind of correct.

## For infinite regular languages
The pumping lemma boils down to this: A regular language can only be infinite by the Kleene Star `*` operator (the operator that repeats whatever preceeds it 0 or more times). The thing preceeding the `*` is the thing that can be repeated in order to "pump" the string.  

If the language is infinite but doesn't have some memoryless infinitely-repeatable section (say its defined as $$\{0^n1^n \vert n \geq 0\}$$) then it can't be regular.

## For infinite context-free languages
A context-free language can be infinite if and only if it has some symbol that derives itself (possibly along with some other stuff). This picture from Wikipedia illustrates the idea pretty well with a derivation tree: ![Pumping lemma derivation tree](https://upload.wikimedia.org/wikipedia/commons/thumb/8/87/Pumping_lemma_for_context-free_languages.svg/612px-Pumping_lemma_for_context-free_languages.svg.png "Pumping lemma derivation tree").

The classic example of a non context-free language is $$\{0^n1^n2^n \vert n \geq 0\}$$. This cannot be produced by any CFG with recursive production rules. What you can do with a CFG is build up sequences with rules like $$R \rightarrow 0R1 \vert \epsilon$$ where the number of "0"s is the same as the number of "1"s. But, you've only got two sides of $$R$$ to work with, so you can't try to throw in a "2" and try to have the same number of "2"s as you do "0"s and "1"s.  

Hopefully this helps. Of course, this is all totally informal so you can't use any of it in a proof. Now go do your homework.