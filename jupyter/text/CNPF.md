# Conditional NPF

## Overview

```{figure} ../images/graph_model_CNPF.svg
---
width: 300em
name: graph_model_CNPF
---
Graphical model for member of the Conditional NPF.
```

The key design choice of members of the NPF is how to model the predictive distribution $p( \mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})$.
In particular, we require a design of the predictive that results in _target set consistency_, i.e., forms a proper stochastic process as discussed in the {ref}`previous section <target_consistency>`.
Arguably the simplest way to assure this property is by assuming that the predictive distribution *factorises* conditioned on the context set (illustrated via graphical representation in {numref}`graph_model_CNPF`).
This means that, having observed the context set $\mathcal{C}$, the prediction for each target location is _independent_ of any other target locations.
We can concisely express this assumption as:

```{math}
---
label: conditional_predictives
---
\begin{align}
p( \mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C}) = \prod_{t=1}^{T} p \left( y^{(t)} | x^{(t)}, \mathcal{C} \right).
\end{align}
```

```{admonition} Advanced$\qquad$Factorisation $\implies$ consistency
---
class: hint, dropdown
---
Recall that the consistency we require means consistency under _marginalisation_ and _permutation_.
It is easy to see that factorisation satisfies consistency under permutation, since for any collection of variables $\{ y_i | i=1,...,N \}$

$$
\begin{align}
  p(y_1, ..., y_N) = \prod_{i=1}^N p(y_i),
\end{align}
$$

and the product operation is commutative.
To see that factorised distributions satisfy consistency under marginalisation, consider the case with two variables, $y_1$ and $y_2$.
Without loss of generality, we have that

$$
\begin{align}
  \int p(y_1, y_2) \mathrm{d}y_2 = \int p(y_1) p(y_2) \mathrm{d}y_2 = p(y_1) \int p(y_2) \mathrm{d} y_2 = p(y_1).
\end{align}
$$

Of course, the same idea works with collections of any size, and marginalising any subset of the variables.
```


A typical (though not necessary) choice is to consider Gaussian distributions for these likelihoods.
We collectively refer to members of the NPF that employ the factorisation assumption as _conditional_ NP models, and to this sub-family as the CNPF.
Now, recall that one guiding principle of the NPF is to _locally_ encode each input-output pair in $\mathcal{C}$, and then _aggregate_ the encodings into a single representation of the context set, which we denote $R$.
Putting these together, we can express the predictive distribution of CNPF members as

```{math}
:label: formal
\begin{align}
p_{\boldsymbol\theta}(\mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})
&= p_{\boldsymbol\theta}(\mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, R) & \text{Parameterisation}  \\
&= \prod_{t=1}^{T} p_{\boldsymbol\theta}(y^{(t)} |  x^{(t)}, R)  & \text{Factorisation}\\
&= \prod_{t=1}^{T} \mathcal{N} \left( y^{(t)};  \mu^{(t)}, \sigma^{2(t)} \right) & \text{Gaussian}
\end{align}
```

Where:

$$
\begin{align}
R^{(c)}
&:= \mathrm{Enc}_{\boldsymbol\theta} \left(x^{(c)}, y^{(c)} \right) & \text{Encoding} \\
R
&:= \mathrm{Agg}\left(\{R^{(c)}\}_{c=1}^{C} \right) & \text{Aggregation} \\
(\mu^{(t)},\sigma^{2(t)})
&:= \mathrm{Dec}_{\boldsymbol\theta}(x^{(t)},R) & \text{Decoding}  
\end{align}
$$

CNPF members make an important tradeoff.
On one hand, we have placed a severe restriction on the class of models that we can fit.
As discussed at the end of this chapter, this restriction has important consequences such as the inability to produce coherent samples.
On the other hand, the factorisation assumption makes evaluation of the predictive likelihoods tractable.
This means that we can employ simple and exact maximum-likelihood procedures to train the model parameters, i.e., training amounts to directly minimising the negative log likelihood ({eq}`training`) which we previously discussed.

```{admonition} Advanced$\qquad$Unbiasedness
---
class: hint, dropdown
---
NOT SURE WHAT WE HAVE PLANNED FOR THIS BOX :)
```

Members of CNPF are distinguished by how they parameterise each of the key components:
* the encoding function $\mathrm{Enc}_{\boldsymbol\theta}$,
* the aggregation function $\mathrm{Agg}$, and
* the decoder $\mathrm{Dec}_{\boldsymbol\theta}$.

Next, we detail some prominent members of the CNPF, and discuss some of their advantages and disadvantages.

(cnp)=
## Conditional Neural Process (CNP)

Arguably the simplest member of the CNPF, and the first considered in the literature, is the Conditional Neural Process (CNP) {cite}`garnelo2018conditional`.
The CNP is defined by the following design choices:
* The encoding function is a feedforward multi-layer perceptron (MLP) that takes as input the concatenation of $x^{(c)}$ and $y^{(c)}$, and outputs a vector $R^{(c)} \in \mathbb{R}^{dr}$.
*  The aggregation function is a simple average over each local representation $\frac{1}{C} \sum_{c=1}^{C} R^{(c)}$.
* The decoder $d_{\boldsymbol\theta}$ is a MLP that takes in the concatenation of the global representation $R$ and the target location $x^{(t)}$, and outputs a mean $\mu^{(t)}$ and standard deviation $\sigma^{(t)}$.

```{admonition} Note$\qquad$Implementation Detail
---
class: note, dropdown
---
Typically, we parameterise $\mathrm{Dec}_{\boldsymbol\theta}$ as outputting $(\mu^{(t)}, \log \sigma^{(t)})$, i.e., the _log_ standard deviation, so as to ensure that no negative variances occur.
```

The resulting computational graph is illustrated in {numref}`computational_graph_CNPs_text`. Notice that the computational cost of making predictions for $T$ target points conditioned on $C$ context points with this design is $\mathcal{O}(T+C)$.
Indeed, each $(x,y)$ pair of the context set is encoded independently ($\mathcal{O}(C)$) and the representation $R$ can then be re-used for predicting at each target location ($\mathcal{O}(T)$).
This means that once trained, CNPs are much more efficient than GPs (which scale as $\mathcal{O}((C+T)^3)$).


```{figure} ../images/computational_graph_CNPs.svg
---
width: 400em
name: computational_graph_CNPs_text
alt: Computational graph CNP
---
Computational graph for CNPs.
```

```{admonition} Advanced$\qquad$Universality and DeepSets
---
class: dropdown, tip
---
There is a strong relationship between this encoder+aggregator and a class of models known as DeepSets networks introduced by Zaheer et al.{cite}`zaheer2017deep`.
In particular, Zaheer et al. show that _any_ continuous function operating on sets ($X$) has a representation of the following form:

$$
\begin{align}
  f(X) = \rho \left( \sum_{x \in X} \phi(x) \right),
\end{align}
$$

for appropriate functions $\rho$ and $\phi$.
To highlight the similarities, we can express each of the mean and standard deviation functions of the CNP as

$$
\begin{align}
  f_\mu \left( x^{(t)}; \mathcal{C} \right) $= \rho_\mu \left( \left[x^{(t)}; \frac{1}{|\mathcal{C}|} \sum_{(x, y) \in \mathcal{C}} \phi(x)\right] \right), \\
  f_\sigma \left( x^{(t)}; \mathcal{C} \right) &= \rho_\sigma \left( \left[x^{(t)}; \frac{1}{|\mathcal{C}|} \sum_{(x, y) \in \mathcal{C}} \phi(x)\right] \right)
\end{align}
$$

where $[\cdot ; \cdot ]$ is the concatenation operation.
In fact, the only difference between the CNP form and the DeepSets of Zaheer et al. is the concatenation operation in $\rho$, and the use of the mean for aggregation, rather than summation.
It is possible (though we do not demonstrate this here) to leverage this relationship to show that CNPs can recover any continuous mean and variance functions, which provides justification for the proposed architecture.

```

Let's see what prediction with such a model looks like in practice.
We first consider a simple 1D setting with samples from a GP with a radial basis function (RBF) kernel (data details in {doc}`Datasets Notebook <../reproducibility/Datasets>`).
Throughout the tutorial, we refer to similar experiments (though we vary the kernel) quite often.
Besides providing useful (and aesthetically pleasing) visualisations, the GPs admit closed form posterior predictive distributions, which allow us to compare to the "best possible" distributions for a given context set.
In particular, if the CNP was "perfect", it would exactly match the predictions of the oracle GP.

```{figure} ../gifs/CNP_rbf.gif
---
width: 35em
name: CNP_rbf_text
alt: CNP on GP with RBF kernel
---
Posterior predictive of a CNP (the blue line represents the predicted mean function and the shaded region shows a standard deviations on each side $[\mu-\sigma,\mu+\sigma]$) and the oracle GP (green line represents the ground truth mean and the dashed line show a standard deviations on each side) with RBF kernel.
```

{numref}`CNP_rbf_text` provides the predictive distribution for a CNP trained on many samples from such a GP.
The figure demonstrates that the CNP performs quite well in this setting.
As more data is observed, the predictions become tighter, as we would hope.
Moreover, we can see that the CNP predictions quite accurately track the ground truth predictive distribution.

That being said, we can see some signs that resemble underfitting: for example, the predictive mean does not pass through all the context points, despite there being no noise in the data-generating distribution.
The underfitting becomes abundantly clear when considering more complicated kernels, such as a periodic kernel (i.e. GPs generating random periodic functions as seen in {ref}`Datasets Notebook <samples_from_gp>`).
One thing we can notice about the ground truth GP is that it leverages the periodic structure, and  becomes confident about predictions at every period.

```{figure} ../gifs/CNP_periodic.gif
---
width: 35em
name: CNP_periodic_text
alt: CNP on GP with Periodic kernel
---
Posterior predictive of a CNP (Blue line for the mean with shaded area for $[\mu-\sigma,\mu+\sigma]$) and the oracle GP (Green line for the mean with dotted lines for +/- standard deviation) with Periodic kernel.
```

Here we see that the CNP completely fails to model the predictive distribution: the mean function is overly smooth and hardly passes through the context points.
Moreover, we might hope that a CNP trained on periodic samples might learn to leverage this structure, but it seems that no notions of periodicity have been learned.
Finally, the uncertainty seems constant, and is significantly overestimated everywhere.
It thus seems reasonable to conclude that the CNP is not expressive enough to accurately model this (more complicated) process.

One potential solution, motivated by the fact that we know CNPs should be able to approximate _any_ mean and variance functions, might be to simply increase capacity of networks $\mathrm{Enc}_{\boldsymbol\theta}$ and $\mathrm{Dec}_{\boldsymbol\theta}$.
Unfortunately, it turns out that CNPs' modelling power scales quite poorly with the capacity of its networks.
A more promising avenue, which we explore next, is to consider the inductive biases of its architectures.


```{admonition} Note
---
class: note
---
Model details and (many more) plots, along with code for constructing and training CNPs, can be found in {doc}`CNP Notebook <../reproducibility/CNP>`
```

(attncnp)=
## Attentive Conditional Neural Process (AttnCNP)


One possible explanation for CNP's underfitting is that all points in the target set share a global representation $R$ of the context set, i.e., $R$ is *independent* of the location of the desired target.
This implies that all points in the context set have the same "importance", regardless of the location at which a prediction is being made.
For example, CNPs struggle to take advantage of the fact that if a target point is very close to a context point they will often both have similar values. A possible solution is to use a target-specific representation $R^{(t)}$, as illustrated in {numref}`computational_graph_AttnCNPs_text`.

```{figure} ../images/computational_graph_AttnCNPs.svg
---
width: 300em
name: computational_graph_AttnCNPs_text
alt: Computational graph of AttnCNP
---
Computational graph for AttnCNPs.
```

```{admonition} Note$\qquad$Functional Representation
---
class: dropdown, note
---
Another perspective, which we will be useful later on, is that the representation $R$ is actually a function of the form $R : \mathcal{X} \to \mathbb{R}^{dr}$ instead of a vector.
This function will be queried at the target position $x^{(t)}$ to yield a target specific vector representation $R^{(t)}$.
```

To achieve this, Kim et al. {cite}`kim2019attentive` propose the Attentive CNP (AttnCNP[^AttnCNP]), which aggregates all the local $R^{(c)}$ in a target dependent representation $R^{(t)}$ using an _attention mechanism_ {cite}`bahdanau2014neural` instead of CNPs' mean aggregation.
There are many great resources available about the use of attention mechanisms in machine learning (e.g. [Distill's interactive visualisation](https://distill.pub/2016/augmented-rnns/#attentional-interfaces), [Lil'Log](https://lilianweng.github.io/lil-log/2018/06/24/attention-attention.html), or the [Illustrated Transformer](http://jalammar.github.io/illustrated-transformer/)), and we encourage readers unfamiliar with the concept to look through these.
For our purposes, it suffices to think of attention mechanisms as learning to _attend_ to specific parts of an input that are particularly relevant to the desired output, giving them more _weight_ than others when making a prediction.

In the AttnCNP, we can think of the attention mechanism as a
set of weights for $\{ w^{(c)}(x^{(t)}) \}$, one for each point in the context set.
Importantly, this set of weights is different for (and depends directly on) every target location!
In other words, the AttnCNP replaces CNP's simple average by a (more general) weighted average which gives a larger weight to "important" context points.


```{admonition} Note$\qquad$Permutation Invariance
---
class: dropdown, note
---
To see that attention mechanisms satisfy permutation invariance, we must provide a more explicit form for the resulting representations.
With an attention mechanism, we can express these forms as

$$
\begin{align}
R^{(t)} = \frac{1}{\sum_{c=1}^{C} w_{\boldsymbol \theta}(x^{(c)}, x^{(t)})} \sum_{c=1}^{C} w_{\boldsymbol \theta}(x^{(c)}, x^{(t)}) R^{(c)},
\end{align}
$$

where $w_{\boldsymbol \theta}(x^{(c)}, x^{(t)})$ is the weight output by the attention mechanism, and $R^{(c)} = \mathrm{Enc}_{\boldsymbol \theta}(x^{(c)}, y^{(c)})$ as before.
Thus, we can see that the resulting representation is simply a weighted average, which is permutation invariant due to the commutativity of the summation operation.
```

To illustrate how attention alleviates underfitting, imagine that our context set contains two points which are "very far" apart in $X$-space.
When making predictions close to the first point, we should largely ignore $R^{(2)}$, since it contains little information about this region of $X$-space compared to $R^{(1)}$.
Attention allows us to parameterise and generalise this intuition, and learn it directly from the data!

This gain in expressivity comes at the cost of increased computational complexity, from  $\mathcal{O}(T+C)$ to $\mathcal{O}(T*C)$, as a representation of the context set now needs to be computed for each target point.

```{admonition} Warning$\qquad$Self-Attention
---
class: dropdown, warning
---
In the above discussion we only consider an attention mechanism between a target and the context set (referred to as cross-attention).
In reality, AttnCNP often first uses an attention mechanism between context points (self-attention) to better model dependencies in the context set.

Notice that this does not impact the permutation invariance as the resulting encoder+aggregator is a composition of permutation invariant functions, which is also permutation invariant.

Using a self-attention layer will also increase the computational complexity to $\mathcal{O}(C(C+T))$ as each context points will first attend to one another in $\mathcal{O}(C^2)$ before the cross attention layer which scales in $\mathcal{O}(C*T)$.
```

To summarise, the AttnCNP is defined by the following design choices:
* The encoding function is either an MLP (as with the CNP), or a _self-attention_ layer operating on the concatenation of $x^{(c)}$ and $y^{(c)}$, outputting vectors $R^{(c)} \in \mathbb{R}^{dr}$.
*  The aggregation function is defined by a cross-attention mechanism, which we can express as

$$
\begin{align}
R^{(t)} = \sum_{c=1}^{C} \tilde{w}_{\boldsymbol \theta} \left( x^{(c)}, x^{(t)} \right) R^{(c)},
\end{align}
$$

where $\tilde{w}$ are the _normalised_ attention weights.
* The decoder $d_{\boldsymbol\theta}$ is a MLP that takes in the concatenation of the target-point specific representation $R^{(t)}$ and the target location $x^{(t)}$, and outputs a mean $\mu^{(t)}$ and standard deviation $\sigma^{(t)}$.


Without further ado, let us see how the AttnCNP performs in practice.
We will first evaluate it on GPs from different kernels (RBF, periodic, and Noisy Matern).

```{figure} ../gifs/AttnCNP_single_gp.gif
---
width: 35em
name: AttnCNP_single_gp_text
alt: AttnCNP on GPs with RBF, Periodic, Noisy Matern kernel
---

Posterior predictive of AttnCNPs (Blue line for the mean with shaded area for $[\mu-\sigma,\mu+\sigma]$) and the oracle GP (Green line for the mean with dotted lines for +/- standard deviation) with RBF, Periodic, and Noisy Matern kernel.
```

{numref}`AttnCNP_single_gp_text` demonstrates that, as desired, AttnCNP alleviates many of the underfitting issues of the CNP, and generally performs much better on the challenging kernels.
However, looking closely at the resulting fits, we can still see some dissatisfying properties:
* The fit on the Periodic kernel is still not _great_.
In particular, we see that the mean and variance functions of the AttnCNP often fail to track the oracle GP, as they do not really leverage the periodic structure in the data.
* Moreover, the posterior predictive of the AttnCNP has "kinks", i.e., it is not very smooth. Notice that these kinks usually appear between 2 context points. This leads us to believe that they are a consequence of the AttnCNP abruptly changing its attention from one context point to the other.

Overall, AttnCNP performs quite well in this setting.
Next, we turn our attention (pun intended) to a more realistic setting, where we do not have access to the underlying data generating process: images.
In our experiments, we consider images as functions from the 2d integer grid (denoted $\mathbb{Z}^2$) to pixel space (this can be grey-scale or RGB, depending on the data).

```{figure} ../images/images_as_functions.png
---
width: 30em
name: images_as_functions_text
---
Viewing images as functions from $\mathbb{Z}^2 \to \mathbb{R}$. Figure from {cite}`garnelo2018neural`.
```

When thinking about images as functions, it makes sense to reference the underlying stochastic process that generated them, though of course we cannot express this process, nor have access to its ground truth posterior predictive distributions.
Just as in the 1D case, our goal is to model the posterior distribution of target pixels (typically the complete image) given a few context pixels.

```{figure} ../gifs/AttnCNP_img_interp.gif
---
width: 30em
name: AttnCNP_img_interp_text
alt: AttnCNP on CelebA and MNIST
---
Posterior predictive of an AttnCNP for CelebA $32\times32$ and MNIST.
```

{numref}`AttnCNP_img_interp_text` illustrates the performance of the AttnCNP on image reconstruction tasks with CelebA (left) and MNIST (right).
The results are quite impressive, as the AttnCNP is able to learn complicated structure in the underlying process, and even produce descent reconstructions of structured obfuscation of the image that is different from the random context pixels seen during training.
The image experiments hammer home an important advantage of the NPF over other approaches to modelling stochastic processes:
the same architecture can scale to complicated underlying processes, learning the important properties from the data (rather than requiring intricate kernel design, as in the case of GPs).
Intuitively, we can think of NPF models as implicitly learning these properties, or kernels, from the data, while allowing us to bake in some inductive biases into the architecture of the model.


```{admonition} Note
---
class: note
---
Model details, training and many more plots in {doc}`AttnCNP Notebook <../reproducibility/AttnCNP>`
```

(extrapolation)=
## Generalisation and Extrapolation

So far, we have seen that well designed CNPF members can flexibly model a range of stochastic processes by being trained from functions sampled from the desired SP.
Next, we consider the question of _generalisation_ and _exptrapolation_ with CNPF members.

One property of GPs is that they can condition predictions on data observed in any region of $X$-space.
A possible downside from the fact that neural processes learn how to model a stochastic process from a dataset, is that during training we must specify a bounded range of $X$ on which data is observed, since we can never sample data from an infinite range.
We know that neural networks are typically quite bad at generalising outside the training distribution, and so we might suspect that CNPF models will not exhibit this appealing property.

Let's first probe this question in the GP experiments.
To do so, we can examine what happens when trained models are conditioned on observations from the underlying process, but outside the training range.


```{figure} ../gifs/CNP_AttnCNP_rbf_extrap.gif
:width: 35em
:name: CNP_AttnCNP_rbf_extrap_text
:alt: extrapolation of CNP on GPs with RBF kernel

Extrapolation of posterior predictive of CNP (Top) and AttnCNP (Bottom) and the oracle GP (Green) with RBF kernel. Left of the red vertical line is the training range, everything to the right is the "extrapolation range".
```

{numref}`CNP_AttnCNP_rbf_extrap_text` clearly shows that the CNP and the AttnCNP  break as soon as the context set contains observations from outside the training range.
In other words, they are not able to model the fact that the RBF kernel is stationary, i.e., that the absolute position of target points is not important only their relative position compared to context points.
Interestingly, they both fail in different ways: the CNP seems to fail for any target location, while the AttnCNP fails only when the target locations are in the extrapolation regime (it still performs well in the training regime).

We can also observe this phenomenon in the image setting.
For example let us evaluate the CNP and AttnCNP on Zero Shot Multi MNIST (ZSMM) where the training set consists of MNIST examples (augmented with translations) while the test images are larger canvases with 2 digits (training and testing examples are shown in the {doc}`Datasets Notebook <../reproducibility/Datasets>`)).


% The following puts 2 figures side by side in large screens but one after the other on phones

````{panels}
:container: container-fluid
:column: col-lg-6 col-md-6 col-sm-12 col-xs-12 p-0 m-0
:card: shadow-none border-0

```{figure} ../gifs/CNP_img_extrap.gif
:width: 100%
:name: CNP_img_extrap_text
:alt: CNP on ZSMM

Posterior predictive of an CNP for ZSMM.
```

---

```{figure} ../gifs/AttnCNP_img_extrap.gif
:width: 100%
:name: AttnCNP_img_extrap_text
:alt: AttnCNP on ZSMM

Posterior predictive of an AttnCNP for ZSMM.
```

````

Again we see in {numref}`CNP_img_extrap_text` and {numref}`AttnCNP_img_extrap_text` that the models completely break in this generalisation task.
This is likely not surprising to anyone who has worked with neural nets as the test set is significantly different of the training set, which represents a challenging setting.
Despite the challenging nature of the failure mode, it turns out that we can in fact construct NPF members that avoid it.
This leads us to our next CNPF member -- the ConvCNP.


(convcnp)=
## Convolutional Conditional Neural Process (ConvCNP)

```{admonition} Caution
---
class: warning
---
The authors of this tutorial are co-authors on the ConvCNP paper.
```

It turns out that the type of generalisation we are looking for -- that the predictions of NPF members depend on the relative position in $X$-space of context and target points rather than the absolute one --
can be mathematically expressed as a property called _translation equivariance_.
Intuitively, translation equivariance (TE) states that if observations are shifted in time or space ($X$-space), then the resulting predictions should be shifted by the same amount.
This simple inductive bias, when appropriate, is _wildly_ effective.
For example, convolutional neural networks (CNNs) were explicitely built to satisfy this property {cite}`fukushima1982neocognitron`{cite}`lecun1989backpropagation` to improve performance and parameter efficiency of neural networks.

In {numref}`translation_equivariance_text`, we visualise translation equivariance in our setting.
Here, we are using a GP with a particular kernel that maintains translation equivariance.
We can see that as the context set is _translated_, i.e., all the data points in $\mathcal{C}$ are shifted by the same amount, so is the resulting predictive distribution.
Intuitively, this property is what we desire when we think about translation equivariant NPF-members.

```{figure} ../gifs/translation_equivariance.gif
:width: 35em
:name: translation_equivariance_text
:alt: Translation equivariant mapping from dataset to predictive

Example of a translation equivariant mapping from a dataset to a predictive stochastic process.
```

```{admonition} Advanced$\qquad$Translation equivariance
---
class: hint, dropdown
---

More precisely, translation equivariance is a property of a mapping between two spaces of functions, or in our case, a mapping between a space of finite datasets and a function space.
Let $\mathcal{Z}$ be our space of datasets, such that each element can be written as $Z = \{(x_n, y_n) | n=1,...,N\}$ for some $N \in \mathbb{N}$, and let $\mathcal{H}$ be an appropriate space of functions on $\mathcal{X}$.
We can define _translations_ on each of these spaces as

$$
\begin{align}
T \colon \mathcal{X} \times \mathcal{Z} \to \mathcal{Z}; & \qquad T_\tau Z = \{ (x_n + \tau, y_n) | n=1, ..., N \} \\
T' \colon \mathcal{X} \times \mathcal{H} \to \mathcal{H}; & \qquad T'_\tau h(x) = h( x - \tau).
\end{align}
$$

Then, a mapping $f \colon \mathcal{Z} \to \mathcal{H}$ is translation equivariant if $f(T_\tau Z) = T'_\tau h$ for all $\tau \in \mathcal{X}$ and $Z \in \mathcal{Z}$.
```

This provides the central motivation behind the ConvCNP {cite}`gordon2019convolutional`: how to bake translation equivariance into CNPF members, while preserving other desirable aspects of the models.
It turns out that for TE to even be well defined, the context set $\mathcal{C}$ needs to be represented by a continous function $R(\cdot): X \to \mathbb{R}^{dr}$ rather than a vector in $\mathbb{R}^{dr}$.
We will refer to representations of this form $R(\cdot)$ as _functional representations_.
ConvCNP essentially works by first mapping the context set to a functional representation and then applying a CNN to ensure that the resulting predictions are TE.
The main question then becomes how to map contexts set to functional representations?
The ConvCNP approaches this question keeping the guiding principles of the NPF in mind.
It first maps each _element_ in $\mathcal{C}$ to a function.
We can express these "local" functions as

```{math}
:label: local_functional_encoding
\begin{align}
  g(x'; x^{(c)}, y^{(c)}) = \begin{bmatrix} 1 \\ y^{(c)} \end{bmatrix} \psi \left( x^{(c)} - x' \right).
\end{align}
```

Here, $\psi$ is a function that maps the _distance between_ $x^{(c)}$ and $x'$ to some real value, which we can think of as similar to a _kernel_.
Note that this encoding of $x^{(c)}, y^{(c)}$ is a function, since it returns a different value for every $x'$.
You may note that the above function is _vector valued_, as it first appends a constant 1 to the output $y^{(c)}$, which results in an additional _channel_.
Intuitively, we can think of this additional channel -- referred to as the _density channel_ -- as keeping track of where data was observed in $\mathcal{C}$ --without this additional constant, the model would be unable to distinguish between not having observed data at a particular location $x$, and having observed the point $(x, 0) \in \mathcal{C}$.
This turns out to be important both in practice, as well as in the theoretical understanding of the ConvCNP!

Now, the aggregation function is simply the sum of all of the local embeddings, which results in yet another function!
So, the final form of the functional representation of the ConvCNP can be written simply as

```{math}
:label: functional_encoding
\begin{align}
   R(\cdot; \mathcal{C}) = \sum_{(x^{(c)}, y^{(c)} \in \mathcal{C})} \begin{bmatrix} 1 \\ y^{(c)} \end{bmatrix} \psi(x^{(c)} - x').
\end{align}
```

If you ever plotted a smooth curve (continuous function) from a set of datapoints, chances are you have probably already heard of either [Kernel Density Estimation](https://en.wikipedia.org/wiki/Kernel_density_estimation) or [Kernel Smoothing](https://en.wikipedia.org/wiki/Kernel_smoother) or [Nadaraya–Watson Kernel Regression](https://en.wikipedia.org/wiki/Kernel_regression#Nadaraya%E2%80%93Watson_kernel_regression), where you smooth the datapoints by convolving them with a Gaussian Kernel (other kernels could be used), i.e., taking a local weighted average of each datapoint.
The functional representation of the ConvCNP is very similar to these methods in spirit!


```{admonition} Important
---
class: attention
---
Although the output of a SetConv is an actual function, only the values of the functional representation at a finite number of locations $\{x^{(u)}\}_{u=1}^{U}$ can be stored on your computer for dowstream processing.
We say the resulting function is *discretised* as it is represented by a finite set $\{(x^{(u)}, R(x^{(u)}))\}_{u=1}^{U}$.

This discretisation means that the resulting ConvCNP can only be approximately TE, where the quality of the approximation is controlled by the number of points $U$.
In practice this turns out to not have a detrimental effect on performance.
The upside is that the resulting (discrete) function can be processed by standard CNNs, as these operate on _discrete_ inputs.
```

```{admonition} Advanced$\qquad$ConvDeepSets
---
class: dropdown, hint
---
Before proposing the ConvCNP, Gordon et al. introduce the _ConvDeepSet_ model, which extends DeepSets {cite}`zaheer2017deep` to translation equivariant functions.
ConvDeepSets map sets of the form $Z = \{(x_n, y_n) | n=1,...,N \}$ for some $n \in \mathbb{N}$ to function spaces, and have the following form

$$
\begin{align}
f(Z) = \rho(E (Z)); \qquad E(Z) = \sum_{(x,y) \in Z} \phi(y) \psi(\cdot - x) \in \mathcal{H},
\end{align}
$$

for appropriate functions $\rho$, $\phi$, and $\psi$.
ConvCNPs specify $\phi \colon y \mapsto (1, y)^T$, $\rho$ as a CNN, and $\psi$ as a kernel function, and use ConvDeepSets to parameterise the mean and standard deviation functions for predictive processes.

The key result of Gordon et al. is to demonstrate that not only are functions of this form translation equivariant, but under mild conditions, any continuous and translation equivariant function on sets $Z$ has a representation of this form.
Similar to how DeepSets can be used to provide justification of the CNP architecture, ConvDeepSets motivate the architectural choices of the ConvCNP.
```

After the discretisation, ConvCNPs employ CNNs as decoders, that are applied to the resulting discrete signal.
Finally, following the final layer of the CNN, an additional smoothing channel is used, similar to the functional form of the encoder (but without the density channel), to evaluate the mean and variance functions at any target location $x^{(t)}$.
We can express this final layer as

```{math}
:label: final_decoder_layer
\begin{align}
   \mu^{(t)} = \sum_{i=1}^T f_\mu(t_i) \psi_\rho(x^{(t)} - t_i);
   \quad
   \log \sigma^{(t)} = \sum_{i=1}^T f_\sigma(t_i) \psi_\rho(x^{(t)} - t_i),
\end{align}
```

where $\{t_i\}_{i=1}^T$ are the discretisation locations, $\psi_\rho$ is another "kernel" function employed in this layer, and $f_\mu, f_\sigma$ are the output channels of the CNN for the mean and variance functions, respectively.
Putting everything together, we can define the ConvCNP with the following design choices (illustrated in {numref}`computational_graph_ConvCNPs_text`):
* The local encoding maps context set points to functions via {numref}`local_functional_encoding`.
* The aggregation function is simply the sum of the local functional encodings ({numref}`functional_encoding`), followed by a discretisation to evaluate the resulting function at a fixed grid of locations.
* Finally, the decoder is a CNN, followed by an additional layer to evaluate the means and standard deviations at the target locations ({numref}`final_decoder_layer`).

<!---
Putting everything together, the ConvCNP computes a target-dependent representation $R^{(t)}$ as follows (illustrated in {numref}`computational_graph_ConvCNPs_text`):
1. Map the context set to a functional representation $R(\cdot)$ using SetConv.
2. Discretise $R(\cdot)$.
3. Process the discretised $R(\cdot)$ with a CNN.
4. Get a target dependent representation $R^{(t)}$ by evaluating the functional representation at each target location $R(x^{(t)})$.
-->

```{figure} ../images/computational_graph_ConvCNPs.svg
---
height: 300em
name: computational_graph_ConvCNPs_text
alt: Computational graph ConvCNP
---
Computational graph of ConvCNP.
```


TO DO: ADD GIF OF CONVCNP

<!---
```{admonition} Advanced$\qquad$Representation Theorem
---
class: dropdown, hint
---
{cite}`gordon2019convolutional` demonstrate that _any_ (continuous) permutation invariance and TE function can be represented by a ConvCNP.
The proof relies on first extending the DeepSets work of {cite}`zaheer2017deep` to include TE as well.
In the {doc}`Theory<Theory>` chapter, we provide a sketch of this proof.
```
-->

```{admonition} Note$\qquad$Computational Complexity
---
class: dropdown, note
---
Computing the discrete functional representation amounts in considering $K$ points in the context set for each discrete location which scales in $\mathcal{O}(U*K)$.
Similarly, computing the target location scales in $\mathcal{O}(T*K).
The computational complexity of inference in a ConvCNP is thus $\mathcal{O}(K(U+T))$.

This shows that there is a trade-off: if the number of discrete points $U$ is too large then the computational cost will not be manageable, but if it is too small then ConvCNP will only be approximately TE.

In typical CNNs, $K$ is very small, in which case ConvCNP usually scales better than AttnCNP (with self-attnetion $\mathcal{O}(C(C+T))$) as long as $U$ is not too large
```

Now that we have constructed a translation equivariant CNPF, we can test it in the more challenging extrapolation regime.
We begin with the same set of GP experiments, but this time already including data observed from outside the original training range.

```{figure} ../gifs/ConvCNP_single_gp_extrap.gif
---
width: 35em
name: ConvCNP_single_gp_extrap_text
---
Extrapolation (red dashes) of posterior predictive of ConvCNPs (Blue) and the oracle GP (Green) with (top) RBF, (center) periodic, and (bottom) Noisy Matern kernel.
```

{numref}`ConvCNP_single_gp_extrap_text` demonstrates that the ConvCNP indeed performs very well!
In particular, we can see that:
* Like the use of attention, the TE inductive bias also helps the model avoid the tendency to underfit the data.
* Unlike the other members of the CNPF, the ConvCNP is able to extrapolate outside of the training range.
Note that this is a direct consequence of TE.
* Unlike attention, the ConvCNP produces smooth mean and variance functions, avoiding the "kinks" introduced by the AttnCNP.
* The ConvCNP is able to learn about the underlying structure in the periodic kernel.
We can see this by noting that it produces periodic predictions, even "far" away from the observed data.


````{admonition} Caution$\qquad$Receptive Field
---
class: dropdown, warning
---
The periodic kernel example is a little misleading, indeed the ConvCNP does not recover the underlying GP.
In fact, we know that it cannot exactly recover the underlying process.
Indeed, it can only model local periodicity because it has a _bounded receptive field_ --- the
size of the region of input that affect a particular output.
This is best seen when considering a much larger target interval ($[-2,14]$ instead of $[0,4]$), as below.

```{figure} ../gifs/ConvCNP_periodic_large_extrap.gif
---
width: 35em
name: ConvCNP_periodic_large_extrap_text
alt: ConvCNP on single images
---
Large extrapolation (red dashes) of posterior predictive of ConvCNPs (Blue) and the oracle GP (Green) with periodic kernel.
```

In fact, this is true for any of the GPs above, all of which have infinite receptive fields, meaning that no model with a bounded field (such as the ConvCNP) can ever exactly recover them.
This discussion alludes to one of the key design choices of the ConvCNP, which is the size of its receptive field.
Note that, unlike standard CNNs, the resulting receptive field does not only depend on the CNN architecture, but also on the granularity of the discretisation employed on the functional representation.
Thus, when designing ConvCNP architectures, some consideration should be given to the interplay between the discretisation and the architecture of the decoder.
````


Let us also examine the performance of the ConvCNP on the more challenging image experiments.
As with the AttnCNP, we consider CelebA and MNIST reconstruction experiments, but also unclude the ZSMM experiments that evaluate the model's ability to generalise beyond the training data.

```{figure} ../gifs/ConvCNP_img.gif
---
width: 45em
name: ConvCNP_img_text
alt: ConvCNP on CelebA, MNIST, ZSMM
---

Posterior predictive of an ConvCNP for CelebA, MNIST, and ZSMM.
```

From {numref}`ConvCNP_img_text` we see that the ConvCNP performs quite well on all datasets when the context set is large enough and uniformly sampled, even when extrapolation is needed (ZSMM).
However, performance is less impressive when the context set is very small or when it is structured, e.g., half images.
In our experiments we find that this is more of an issue for the ConvCNP than the AttnCNP ({numref}`AttnCNP_img`), we hypothesize that this happen because the effective receptive field of the former is too small.

```{admonition} Note$\qquad$Effective Receptive Field
---
class: dropdown, note
---
We call the *effective* receptive field, the empirical receptive field of a *trained* model rather than the theoretical receptive field for a given architecture.
For example if a ConvCNP always observes many context points during training, then every target point will be close to some to context points and the ConvCNP will thus not need to learn to depend on context points that are far from target points.
The size of the effective receptive field can thus be incfreased by reducing the size of the context set seen during training, but this solution is somewhat dissatisfying in that it requires tinkering with the training procedure.

Notice that this is not really an issue with AttnCNP which both always has to attend to all the context points, i.e., it has ``infinite'' receptive field.
```

Although the previous plots look good, you might wonder how such a model compares to standard interpolation baselines.
To answer to this question we will look at larger images to see the more fine grain details.
Specifically, let us consider a ConvCNP trained on $128 \times 128$ CelebA:

```{figure} ../gifs/ConvCNP_img_baselines.gif
---
width: 25em
name: ConvCNP_img_baselines
alt: ConvCNP and baselines on CelebA 128
---

ConvCNP and Nearest neighbour, bilinear, bicubic interpolation on CelebA 128.
```

From {numref}`ConvCNP_img_baselines_text` shows that ConvCNP performs much better than standard interpolation methods.
These results are very encouraging.
Having seen such encouraging results, as well as the descent zero shot generalization capacity of ConvCNP on an artificial dataset (ZSMM), it is natural to want to evaluate the model on actual images with multiple faces of different scale and orientation.

```{figure} ../images/ConvCNP_img_zeroshot.png
---
width: 20em
name: ConvCNP_img_zeroshot_text
alt: Zero shot generalization of ConvCNP to a real picture
---
Zero shot generalization of a ConvCNP trained on CelebA and evaluated on Ellen's selfie. We also show a baseline bilinear interpolator.
```

From {numref}`ConvCNP_img_zeroshot_text` we see that the model is able to generalise reasonably well to real world data in a zero shot fashion.
Although the previous results look nice, the use-cases are not immediately obvious, as it is not very to have missing pixels.
One possible application is increasing the resolution of an image.
This can be achieved by querying positions "in between" pixels.

```{figure} ../images/ConvCNP_superes_baseline.png
---
width: 30em
name: ConvCNP_superes_baseline_text
alt: Increasing image resolution with ConvCNP
---
Increasing the resolution of $16 \times 16$ CelebA to $128 \times 128$ with a ConvCNP and a baselin bilinear interpolator.
```

{numref}`ConvCNP_superes_baseline_text` demonstrates such an application.
We see that CNPFs can indeed be used to increase the resolution of an image better than the standard bilinear interpolator, even though it was not trained to do so!


```{admonition} Note
---
class: note
---
Model details, training and many more plots are available in the {doc}`ConvCNP Notebook <../reproducibility/ConvCNP>`
```

(issues-cnpfs)=
## Issues With CNPFs

Let's take a step back.
So far, we have seen that we can use the factorisation assumption to construct simple members of the CNPF, perhaps the simplest of these being the CNP.
Our first observation was that while the CNP can track underlying processes, it tends to underfit when the processes are more complicated.
We saw that this tendency can be addressed by adding appropriate inductive biases into the paramterisation of the model.
Specifically, the AttnCNP significantly improves upon the CNP by adding an attention mechanism to generate target-specific representations of the context set.
We further saw that the AttnCNP also scales nicely to image-settings.
However, both the CNP and AttnCNP fail to make meaningful predictions when data is observed outside the training range (or when the test distribution is different from training, i.e., in the ZSMM example).
Finally, we saw how including TE as an inductive bias led to well-fitting functions that generalised elegantly to observations outside the training range.

Let us now consider more closely the implications of the factorisation assumption, along with the Gaussian form of predictive distributions.
One immediate consequence of using the Gaussian likelihood is that we cannot recover multi-modal distributions.
To see why this might be an issue, consider making predictions for the MNIST reconstruction experiments.

```{figure} ../images/ConvCNP_marginal.png
---
width: 20em
name: ConvCNP_marginal_text
alt: Samples from ConvCNP on MNIST and posterior of different pixels
---

Posterior predictive of ConvCNPs on an entire MNIST image (left) and posterior predictive of some pixels (right).
```

Looking at {numref}`ConvCNP_marginal_text`, we might expect that sampling from the predictive distribution of an unobserved pixels sometimes yield completely white values, and sometimes completely black, depending on whether the sample represents, for example, a 3 or a 5.
However, a Gaussian distribution, which is uni-modal (see {numref}`ConvCNP_marginal_text` right), cannot achieve this type of multi-modal behaviour.

```{admonition} Note
---
class: note, dropdown
---
One solution to this particular problem might be to employ some other parametric distribution that enables multimodality, for example, a mixture of Gaussians.
While this may solve some issues, we can generalise this point to say that the CNPF requires specifying _some parametric form of distribution_.
Ideally, what we would like is some parametrisation of the NPF that enables us to recover _any_ form of marginal distribution.
```

The other major restriction is the factorisation assumption itself, which has several important implications.
First, it means that the model can not leverage any correlation structure that might exist in the predictive distribution over multiple target sets.
For example imagine that we are modelling samples from an underlying GP.
If the model is making predictions at two target locations that are "close" in $X$-space, it seems reasonable that whenever it predicts the first be "high", it predict something similar for the second, and vice versa.
Yet the factorisation assumption means that the model cannot learn this type of structure.
Another implication is that we can not produce _coherent_ samples from the predictive distribution.
In fact, sampling from the posterior corresponds to adding independent noise to the mean at each target location, resulting in samples that are discontinuous and look nothing like the underlying process.

```{figure} ../images/ConvCNP_rbf_samples.png
---
width: 35em
name: ConvCNP_rbf_samples_text
alt: Sampling from ConvCNP on GP with RBF kernel
---
Samples form the posterior predictive of ConvCNPs (Blue) and the oracle GP (Green) with RBF kernel.
```

Similary, sampled images from a member of the CPNF are not coherent and look like arbitrary noise added to a picture.

```{figure} ../images/ConvCNP_img_sampling.png
---
width: 45em
name: ConvCNP_img_sampling_text
alt: Sampling from ConvCNP on CelebA, MNIST, ZSMM
---

Samples from the posterior predictive of an ConvCNP for CelebA, MNIST, and ZSMM.
```


```{admonition} Note
---
class: note, dropdown
---
This inability to sample from the predictive may inhibit the deployment of CNPF members from several application areas for which it might otherwise be potentially well-suited.
One such example is the use of Thompson sampling algorithms for e.g., contextual bandits or Bayesian optimisation, which require a model to produce samples.
```

In the next chapter, we will see one approach to solving both these issues by treating the representation as a latent variable in the latent Neural Process sub-family.

[^AttnCNP]: {cite}`kim2019attentive` only introduced the latent variable model, but one can easily drop the latent variable if not needed.
