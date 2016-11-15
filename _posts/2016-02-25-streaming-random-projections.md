---
layout: post
title: Online Random Projections
tags: research
---

![feature](/images/streamhash-feature.jpg)
I recently encountered the problem of computing the similarity of pairs of
"documents" (which, in my case, were actually graphs), where the documents
arrived as a stream of individual words. The incoming stream of words could both
initiate new documents and grow existing documents. Interestingly, the words of each
document could also arrive out-of-order, but I will ignore this complication for now. 


The problem then was to continuously compute and maintain the similarity of 
every pair of documents originating from the words in the stream. Strangely, I
could not find any existing methods to do this even for the case of a single
document pair, under the constraints of the data stream model (single-pass,
bounded memory). Specifically, all the methods I surveyed needed to know
my "vocabulary": the unique words that could possibly arrive in the stream.
In my case, this is never known.

In this post, I will describe the method I developed to tackle this problem,
while also discussing the background needed to understand why it works. The
method itself is fairly simple and, with some help from the code snippets in this
post, should be easy to implement and try out yourself!

### Contents
{:.no_toc}

* TOC
{:toc}

## Computing Document Similarity 

![Cosine Distance](https://docs.google.com/drawings/d/1pypq-JElxpCNn12brQ1eW9XzD2NA0_td7aZGv7S6eyM/pub?w=1200)

***Computing the cosine similarity of documents D1 and D2:** The documents are
first split into their constituent words and a vocabulary index is assigned to each
word. The documents are then represented by vectors of frequencies of the words they
contain. Given these vectors, their magnitudes, dot-product and cosine-similarity
can be computed.*
{:.caption}

A common way to measure how similar two documents are is by their
[cosine similarity][1]. An example of this is illustrated
in the figure above. All documents in the collection are first split into
their constituent words, and a *vocabulary index* is constructed which assigns
an integer to each unique word. If the vocabulary size is $$|V|$$, each document is
represented by a $$|V|$$-sized vector of frequencies of the words it contains.

Given these vectors for a pair of documents, say **D1** and **D2**, their
magnitudes are given by,

$$
\begin{align*}
  magn(\mathbf{D1}) &= \|\mathbf{D1}\|_2\\
  magn(\mathbf{D2}) &= \|\mathbf{D2}\|_2,
\end{align*}
$$

and their cosine similarity is given by,

$$
  sim(\mathbf{D1}, \mathbf{D2}) = cos(\mathbf{D1}, \mathbf{D2})
                                = \frac{\mathbf{D1} \cdot \mathbf{D2}}
                                       {\|\mathbf{D1}\|_2\|\mathbf{D2}\|_2}.
$$

### A Computational Problem With Large |V|
{:.no_toc}

In practice, it is possible for the
vocabulary to be extremely large. This increases the time needed to compute the
similarity of each document pair and the memory needed to store each document.
To address this issue, a popular approach is to *sketch* the word-frequency vectors
that represent each document. Sketching replaces $$|V|$$-sized document vectors by
$$k$$-sized *sketch* vectors, where $$k$$ is much smaller than $$|V|$$, while accurately
approximating the similarity of pairs of documents.

## Sketching Document Similarity 

An approach to sketching vectors that preserves their cosine similarity is using
[random projections][2], which applies some neat tricks from linear algebra and
probability.
Let's say we want to compute the similarity of documents with vectors **D1** and
**D2** having angle θ between them. We can visualize these vectors in the XY-plane. 

![Document Vectors](https://docs.google.com/drawings/d/15MsVhJreASVW-VnJYCIYTUGNS93pEysOvwQ1jqsZdaU/pub?w=1200)

Let's pick a vector **R** in the plane having the same size as **D1** and
**D2** uniformly at random in the plane. Then define a function *h* that
operates on any such random vector **R** and a document vector **D** as follows:

$$
  h_{\mathbf{R}}(\mathbf{D}) =
    \begin{cases}
      +1 ~\textrm{if}~ \mathbf{D}\cdot\mathbf{R} \geq 0,\\
      -1 ~\textrm{if}~ \mathbf{D}\cdot\mathbf{R} < 0.
    \end{cases}
$$

So $$ h_{\mathbf{R}}(\mathbf{D1}) = h_{\mathbf{R}}(\mathbf{D2}) $$ only if
both **D1** and **D2** lie on the same side of **R**, and
$$ h_{\mathbf{R}}(\mathbf{D1}) \neq h_{\mathbf{R}}(\mathbf{D2}) $$ otherwise.
Now what is the probability that
$$ h_{\mathbf{R}}(\mathbf{D1}) = h_{\mathbf{R}}(\mathbf{D2}) $$, over all random
choices of **R**? It's easier to see what this is with another illustration.

![Random Vectors](https://docs.google.com/drawings/d/1b7SQPGmNJ3Ig_gRfFD0jsoDIaiLrqCMV5QUwIHBuh0w/pub?w=1200)

In the red region, $$ h_{\mathbf{R}}(\mathbf{D1}) \neq h_{\mathbf{R}}(\mathbf{D2}) $$
since **D1** and **D2** will fall on opposite sides of any random vector **R** chosen
in this region. Similarly,
$$ h_{\mathbf{R}}(\mathbf{D1}) = h_{\mathbf{R}}(\mathbf{D2}) $$ in the yellow region.
The ratio of the areas of the red and yellow regions is *2θ/2π*, leading to the
probability over random vectors **R**:

$$
  P_{\mathbf{R}}[h_{\mathbf{R}}(\mathbf{D1}) =  h_{\mathbf{R}}(\mathbf{D2})]
    = 1 - θ/π.
$$

If we could somehow estimate this probability, we could plug it into the above
equation to find the angle θ, and hence, find the cosine similarity of the two
document vectors:

$$
\begin{gather*}
  θ = π (1 - P_{\mathbf{R}}[h_{\mathbf{R}}(\mathbf{D1}) =
                              h_{\mathbf{R}}(\mathbf{D2})])\\
  sim(\mathbf{D1}, \mathbf{D2}) = cos(θ)
\end{gather*}
$$

It turns out that it's easy to estimate this probability: simply pick $$k$$ random
vectors $$ \mathbf{R}_1, \dots, \mathbf{R}_k $$, evaluate
$$ h_{\mathbf{R_i}}(\mathbf{D1}) $$ and $$ h_{\mathbf{R_i}}(\mathbf{D2}) $$ for
each of these vectors and then count the fraction of function values for
which the two documents agree (i.e. function values are both +1 or both -1),

$$
  P_{\mathbf{R}}[h_{\mathbf{R}}(\mathbf{D1}) = h_{\mathbf{R}}(\mathbf{D2})]
    \approx \frac{\sum_{i=1}^k \mathbb{1}(h_{\mathbf{R_i}}(\mathbf{D1}) =
                                          h_{\mathbf{R_i}}(\mathbf{D2}))}
                 {k}. 
$$

In practice, the random vectors turn out to be sufficiently random if each
of their elements are chosen uniformly from $$\{+1, -1\}$$.

By fixing a set of random vectors $$ \mathbf{R}_1, \dots, \mathbf{R}_k $$,
every $$|V|$$-sized document vector **D** can be replaced by a $$k$$-sized *sketch*
vector, where element $$i$$ of the sketch vector is either +1 or -1 and is given by
$$ h_{\mathbf{R_i}}(\mathbf{D}) $$. Thus, each sketch vector is essentially a bit
vector and consumes very little space. Document similarity can then be estimated
using just these sketch vectors. The accuracy of estimation improves with increasing
$$k$$.

### A Computational Problem With Unknown |V|
{:.no_toc}

Notice that in the random projections technique just described, we need to know
$$|V|$$ to construct $$|V|$$-sized random vectors. In a streaming scenario, $$|V|$$
is never known, and may grow with time! 

## Streaming Document Similarity

Let's say we could pick $$k$$ functions $$h_1, \dots, h_k$$ at random from a
family of functions $$\mathcal{H}$$, where each $$h \in \mathcal{H}$$ maps a word
to an element of $$\{+1, -1\}$$. Then we could replace each random vector
$$\mathbf{R}_i$$ with a function $$h_i \in \mathcal{H}$$, as long as $$\mathcal{H}$$
obeys the following properties:

  1. *It should be equally probable for a given word to map to +1 or -1, over
     randomly chosen $$h \in \mathcal{H}$$.*

     This captures the equivalence of the function $$h_i$$ with $$\mathbf{R}_i$$
     whose elements are each uniformly chosen from $$\{+1, -1\}$$.
     
     However, requiring only this property permits trival families such as
     $$\mathcal{H} = \{h_1(w) = +1, ~h_2(w) = -1, \forall ~\textrm{words}~ w\}$$.
     This family satisfies property (1) since the probability that any function randomly
     picked from $$\mathcal{H}$$ maps a word to +1 or -1 is 1/2; note that the
     randomness originates from picking the function and not from computing the
     mapped value of a word. But each of the functions in this trivial family map
     all words to the same value, which is not very useful.

     So we need the following additional property:

  2. *For a given $$h \in \mathcal{H}$$, values in \{+1, -1\} are equally probable
     over all words.*

     This ensures that a function in the family cannot map all words to the same
     value, and should evenly distribute the mapped values across all words between
     +1 and -1. This still does not prevent families such as
     $$\mathcal{H} = \{h_1(w), ~h_2(w) = -h_1(w), \forall ~\textrm{words}~ w\}$$
     where $$h_1$$ and $$h_2$$ both satisfy property (2). This family is not
     very useful either, since the second function is correlated with the first.

     This leads to our final desired property:

  3. *All the functions in $$\mathcal{H}$$ are pairwise-independent.*

It turns out that a family satisfying these requirements is a
a *2-universal (strongly universal)* hash function family by definition.
Many such families exist that work on strings. One example is the multilinear family:

$$
  h(w) = m_0 + \sum_{i = 1}^n m_i w_i,
$$

where $$w_i$$ is the integer representation of the $$i^\textrm{th}$$ character of
the word $$w$$, and $$m_0, \dots, m_n$$ are uniformly random integers. The
function above returns an integer, but this can be easily mapped to an element of
$$\{+1, -1\}$$ by extracting the MSB and some algebra. 

### Implementation
{:.no_toc}

To sample $$k$$ functions from the strongly-universal multilinear family, it is
only necessary to estimate the maximum word length $$m = |w|_{\textrm{max}}$$.
Then, for each function $$h_1, \dots, h_k$$, $$m + 1$$ random integers are generated
and stored.

The following C++ 11 snippet initializes $$h_1, \dots, h_k$$ as $$k$$ hash functions
sampled uniformly at random from the multilinear family:

{% highlight cpp %}
#include <random>
#define SEED 42

mt19937_64 prng(SEED); /* Mersenne Twister 64-bit PRNG */

/* m = maximum word length, k = number of hash functions */
vector<vector<uint64_t>> H(k, vector<uint64_t>(0, m+1));

for (uint32_t i = 0; i < k; i++) {
  for (uint32_t j = 0; j < m+1; j++) {
    H[i][j] = prng();
  }
}
{% endhighlight %}

Then the mapping for a word $$w$$ using $$h_i$$ can be computed as:

{% highlight cpp %}
/* w is a string of length <= m */
uint64_t sum = H[i][0];
for (uint32_t j = 1; j < m+1; j++) {
  sum += H[i][j] * (static_cast<uint64_t>(word[j-1]) & 0xff);
}
sum = 2 * static_cast<int>((sum >> 63) & 1) - 1;
{% endhighlight %}

Notice the bitwise-and when computing the summation: this is to take care of the
sign-extension performed when a 1-byte character is cast to a 4-byte unsigned
integer.

## Streaming Heterogenous Graph Similarity 

The technique described above was used to perform streaming clustering and anomaly
detection on heterogenous graphs, where the graphs were arriving as a stream of
individual edges. Further details are available in the paper and code at the
[project website][3].

[1]: https://en.wikipedia.org/wiki/Cosine_similarity
[2]: https://en.wikipedia.org/wiki/Random_projection
[3]: http://www3.cs.stonybrook.edu/~emanzoor/streamspot
