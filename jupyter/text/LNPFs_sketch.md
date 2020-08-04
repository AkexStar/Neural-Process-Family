# Latent NPFs

## Overview

We concluded the previous section by noting two important drawbacks of the CNPF:

* The marginal predictive distribution is factorised, and thus can neither account for correlations in the predictive nor (as a result) produce "coherent" samples from the predictive distribution.
* The marginal predictive distributions require specification of a particular parametric form.

In this section we discuss an alternative parametrisation of $p( \mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})$ that addresses both of these issues.
The main idea is to introduce a latent variable $\mathbf{z}$ in to the definition of the predictive distribution.
This leads us to the second major branch of the NPF, which we refer to as the Latent Neural Process Sub-family, or LNPF for short.
A graphical representation of the LNPF  is given in {numref}`graph_model_LNPs_text`.

```{figure} ../images/graph_model_LNPs.svg
---
width: 200em
name: graph_model_LNPs_text
alt: graphical model LNP
---
Graphical model for LNPs.
```

To specify this family of models, we must define a few components:

* An encoder: $p_{\boldsymbol\theta} \left( \mathbf{z} | \mathcal{C} \right)$, which provides a distribution over the latent variable $\mathbf{z}$ having observed the context set $\mathcal{C}$.
* A decoder: $p_{\boldsymbol\theta} \left( y | x, \mathbf{z} \right)$, which provides predictive distributions condition on the latent variable $\mathbf{z}$ and a target location $x$.

The design of the encoder will follow the principles of the NPF, i.e., using local encodings and a permutation invariant aggregation function.
However, here we will use these principles to model a conditional distribution over the latent variable, rather than a deterministic representation.

Again, a typical choice for the decoder is as a Gaussian distribution.
However, as we discuss below, choosing the decoder to have a Gaussian form in this case is far less restrictive than with the CNPF.
With the above components specified, we can now express the predictive distribution as

```{math}
:label: latent_likelihood
\begin{align}
p_{\boldsymbol\theta}(\mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})
&= \int p_{\boldsymbol\theta} \left(\mathbf{y}_\mathcal{T} , \mathbf{z} | \mathbf{x}_\mathcal{T} , \mathcal{C} \right) \mathrm{d}\mathbf{z} & \text{Parameterisation}  \\
&= \int p_{\boldsymbol\theta} \left( \mathbf{z} | \mathcal{C} \right) \prod_{t=1}^{T} p_{\boldsymbol\theta}(y^{(t)} |  x^{(t)}, \mathbf{z}) \mathrm{d}\mathbf{z}  & \text{Factorisation}\\
&= \int p_{\boldsymbol\theta} \left( \mathbf{z} | \mathcal{C} \right)  \prod_{t=1}^{T} \mathcal{N} \left( y^{(t)};  \mu^{(t)}, \sigma^{2(t)} \right) \mathrm{d}\mathbf{z} & \text{Gaussian}
\end{align}
```

Now, you might note that we have still made both the factorisation and Gaussian  assumptions!
However, while the decoder likelihood $p_{\boldsymbol\theta}(\mathbf{y} | \mathbf{x}, \mathbf{z})$ is still factorised, the predictive distribution we are actually interested in -- $p( \mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})$ -- which is defined by _marginalising_ the latent variable $\mathbf{z}$, is no longer factorised, thus addressing the first problem we associated with the CNPF.
Moreover, that distribution, which we refer to as the _marginal predictive_, is no longer Gaussian either.
In fact, by noting that the marginal predictive now has the form of an _infinite mixture of Gaussians_, we can conclude that _any_ predictive distribution can be represented (i.e. learned) by this form.
This is great news, as it (conceptually) relieves us of the burden of choosing / designing an appropriate likelihood when deploying the NPF for a new application!

While this parameterisation seems to solve the major problems associated with the CNPF, it introduces an important drawback.
In particular, the key difficulty with the LNPF is that the likelihood we defined in {numref}`latent_likelihood` is no longer _tractable_.
This has several severe implications, and in particular means that we can longer use simple maximum-likelihood training for the parameters of the model.
In the remainder of this section, we first discuss the  question of training members of the LNPF, without having any particular member in mind.
After discussing several training procedures, we discuss extensions of each of the CNPF members introduced in the previous section to their corresponding member of the LNPF.

## Training LNPF members

Ideally, what we would like to do is use the likelihood defined in {numref}`latent_likelihood` to optimise the parameters of the model.
However, this quantity is not tractable, and so we must consider alternative procedures.
In fact, this story is not new, and the same issues arise when considering many latent variable models, such as variational auto-encoders (VAEs).

The question of training LNPF members is still open, and there is ongoing research in this area.
In this section, we will cover two methods for training LNPF models, but we emphasise that both have their flaws, and deciding on an appropriate training method is an open question that must often be answered empirically.


### Neural Process Variational Inference (NPVI)

The first solution to training LNPF members, proposed by {cite}`garnelo2018neural` takes inspiration from the literature on variational inference (VI), and in particular, amortised VI, which is used in training VAEs.
There are many resources available on amortised VI (LINKS TO BLOGS / PAPERS), and we encourage readers unfamiliar with the concept to take the time to go through these.
Many of the relevant ideas are widely applicable, and provide valuable insights for many sub-areas of ML.
For our purposes, the following intuitions should suffice.

The central idea in amortised VI is to introduce an _inference network_, denoted $q_{\boldsymbol\phi}$, which is trained to approximate the _posterior distribution_ over the latent variable.
In our case, the posterior distribution of interest is $p_{\boldsymbol\theta}(\mathbf{z} | \mathcal{C}, \mathcal{T})$, i.e., the distribution of the latent variable had we observed _both_ the context and target sets.
To approximate this posterior, we can introduce a network that maps datasets to distributions over the latent variable.
We already know how to define such networks in the NPF -- it has the same form as our encoder $p_{\boldsymbol\theta}(\mathbf{z} | \mathcal{C})!
In fact, as we will see below, NPVI proposes to use the encoder as the inference network when training LNPF members.

```{admonition} Advanced
---
class: dropdown, caution
---
In fact, the inference network is trained to approximate a mapping from the observed data to the posterior distribution over the latent variable.
This is where the term _amortised_ comes from: rather than freely optimise the parameters of each approximate posterior distribution, we share the parameters via a global mapping (often parameterised as a neural network).
```

Having introduced $q_{\boldsymbol\phi}$, we can use it to derive a _lower bound_ (often coined an _ELBO_) to the log marginal likelihood we would like to optimise.
Denoting $\mathcal{D} = \mathcal{C} \cup \mathcal{T}$, we have that

```{math}
:label: np_elbo
\begin{align}
p_{\boldsymbol\theta}(\mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})
\geq \mathbb{E}_{\mathbf{z} \sim q_{\boldsymbol\phi}(\mathbf{z} | \mathcal{D})} \left[ \sum_{t=1}^T \log p_{\boldsymbol\theta} \left( y^{(t)} | x^{(t)}, \mathbf{z} \right) \right] - \mathrm{KL} \left( q_{\boldsymbol\phi} \left( \mathbf{z} | \mathcal{D} \right) \| p_{\boldsymbol \theta} \left( \mathbf{z} | \mathcal{C} \right) \right),
\end{align}
```

where $\mathrm{KL}(p \| q)$ is the Kullback-Liebler (KL) divergence between two distributions $p$ and $q$.
We can derive an unbiased estimator to {numref}`np_elbo` by taking samples from $q_{\boldsymbol\phi}$ to estimate the first term on RHS.
When both the encoder and inference network parameterise Gaussian distributions over $\mathbf{z}$ (as is standard), the KL-term can be computed analytically.
Note that {numref}`np_elbo` provides an objective function for training both the inference network parameters $\boldsymbol\phi$ and model parameters $\boldsymbol\theta$.
In fact, typical practice when designing LNPF members is to _share_ the parameters between the encoder and inference network.
When doing so, we can express a single training step for LNPF members with NPVI as follows:

1. Sample a task $(\mathcal{C}, \mathcal{T})$ from the data.
2. Sample $L$ samples as $\mathbf{z}_l \sim p_{\boldsymbol\theta} \left(\mathbf{z} | \mathcal{D} \right)$.
3. Approximate the lower-bound as (assuming the KL has an analytical form)
```{math}
\begin{align}
\hat{\mathcal{L}} \leftarrow \frac{1}{L} \sum_{l=1}^{L} \sum_{t=1}^{T} \log p_{\boldsymbol\theta} \left( y^{(t)} |  x^{(t)}, \mathbf{z}_l \right) - \mathrm{KL} \left( p_{\boldsymbol\theta} \left(\mathbf{z} | \mathcal{D} \right) \| p_{\boldsymbol\theta} \left(\mathbf{z} | \mathcal{C} \right) \right).
\end{align}
```
4. Use the backpropagation algorithm to take a gradient step in $\boldsymbol\theta$ to maximize $\hat{\mathcal{L}}$.


```{admonition} Warning
---
class: caution, dropdown
---
There are several nuances with the derivation of the NPVI ELBO that we have glossed over for the sake of brevity.
In fact, the original derivation of this procedure ({cite}`garnelo2018neural`) invokes slightly different modelling assumptions than what we have used, and ends up introducing an approximation which results in the resulting ELBO not being a proper lower bound on the log-marginal likelihood.
In the {doc}`Theory <Theory>` chapter we discuss in detail the modelling assumptions associated with the derivations, provide a full derivation of the ELBO, and discuss the implications of the approximations that must be made along the way.
```

NPVI inherits several appealing properties from the VI framework:
* It utilises _posterior sampling_ to reduce the variance of the Monte-Carlo estimator of the intractable expectation. This means that often we can get away with training models taking just a single sample, resulting in computationally efficient training procedures.
* If our approximate posterior can recover the true posterior, the inequality is tight, and we are exactly optimising the log-marginal likelihood.

However, it also inherits the main drawbacks of VI.
In particular, it is almost never the case that the true posterior can be recovered by our approximate posterior in any practical application.
In these settings:

* Meaningful guarantees about the quality of the learned models are hard to come by.
* The inequality holds, meaning that we are only optimising a lower-bound to the quantity we actually care about. Moreover, it is often quite difficult to know how tight this bound may be.

Finally, NPVI adds additional drawbacks that are unique to the NPF setting.
These can be roughly summarised as

* VI is heavily focused on approximating the posterior distribution, and diverts encoder capacity and attention of the optimiser to recovering the true posterior. However, in the NPF setting, we are often only interested in the predictive distribution $p(\mathbf{y}_T | \mathbf{x}_T, \mathcal{C})$, and it is unclear whether focusing on the approximate posterior is beneficial to achieving higher quality predictive distributions.
* Sharing the encoder and inference network introduces additional complexities in the training procedure. In particular, it muddies the distinction between modelling and inference that is typical in probabilistic modelling. Moreover, it is unclear what the effect of the dual roles of the encoder in the model are, and it may be that using the encoder as an approximate posterior has a detrimental effect on the resulting predictive distributions.

Despite these (and other) drawbacks NPVI is the most commonly employed procedure for training LNPF members.
Next, we turn our attention to recently proposed procedure that abandons the approximate posterior interpretation of training LNPF members in favour of a simpler, approximate maximum-likelihood procedure.


### Neural Process Maximum Likelihood (NPML)

A more direct approach is to optimise the log-marginal predictive likelihood directly.
We can achieve this by using the log-sum-exp trick, as follows

```{math}
:label: npml
\begin{align}
p_{\boldsymbol\theta}(\mathbf{y}_\mathcal{T} | \mathbf{x}_\mathcal{T}, \mathcal{C})
&= \log \int p_{\boldsymbol\theta} \left( \mathbf{z} | \mathcal{C} \right) \prod_{t=1}^{T} p_{\boldsymbol\theta} \left( y^{(t)} | x^{(t)}, \mathbf{z} \right) \mathrm{d}\mathbf{z} \\
& \approx \log \left( \frac{1}{L} \sum_{l=1}^{L} \prod_{t=1}^{T} p_{\boldsymbol\theta} \left( y^{(t)} | x^{(t)}, \mathbf{z} \right) \right) \\
& = \log \left( \sum_{l=1}^{L} \exp \left(  \sum_{t=1}^{T} \log p_{\boldsymbol\theta} \left( y^{(t)} | x^{(t)}, \mathbf{z} \right) \right) \right) - \log L\\
& = \text{LogSumExp}_{l=1}^{L} \left( \sum_{t=1}^{T} \log p_{\boldsymbol\theta} \left( y^{(t)} | x^{(t)}, \mathbf{z} \right) \right) - \log L
\end{align}
```


## Latent Neural Process (LNP)

```{figure} ../images/graph_model_LNPs.svg
---
width: 200em
name: graph_model_LNPs_text
alt: graphical model LNP
---
Graphical model for LNPs.
```
```{figure} ../images/computational_graph_LNPs.svg
---
width: 300em
name: computational_graph_LNPs_text
alt: Computational graph LNP
---
Computational graph for LNPS. [drop?]
```

The latent neural process {cite}`garnelo2018neural` is the latent counterpart of CNP, i.e. once we have a representatoin $R$ we will pass it through a MLP to predict the mean and variance of the latent representation from which to sample.
See {numref}`graph_model_LNPs_text` for the graphical model and {numref}`computational_graph_LNPs_text` for the computational graph.

```{admonition} Advanced
---
class: dropdown, caution
---
Theoretical gains of using a latent variable
More information in {doc}`Additional Theory <Theory>`
```

Note on heteroskedastic noise which is strange and as a result in the case of modeling GPs it collapses to CNPs if you do not use a lower bound[]...


```{figure} ../gifs/LNP_rbf.gif
---
width: 35em
name: LNP_rbf_text
alt: LNP on GP with RBF kernel
---
Samples from posterior predictive of LNPs (Blue) and the oracle GP (Green) with RBF kernel.
```

{numref}`graph_model_LNPs_text` shows that the latent variable indeed enables coherent sampling from the posterior predictive.
It nevertheless suffers from the same underfitting issue as discussed with CNPs.



```{admonition} Details
---
class: tip
---
Model details, training and many more plots in {doc}`LNP Notebook <../reproducibility/LNP>`
```

## Attentive Latent Neural Process (AttnLNP)

```{figure} ../images/graph_model_AttnLNPs.svg
---
width: 200em
name: graph_model_AttnLNPs_text
alt: graphical model AttnLNP
---
Graphical model for AttnLNPs.
```
```{figure} ../images/computational_graph_AttnLNPs.svg
---
width: 400em
name: computational_graph_AttnLNPs_text
alt: Computational graph AttnLNP
---
Computational graph for AttnLNPS. [drop?]
```

The Attentive LNPs {cite}`kim2019attentive` is the latent counterpart of AttnCNPs. The way they incorporated the latent variable is a little different than other LNPs, in that they added a "latent path" in addition (not instead) of the deterministic path, giving rise to the (strange?) graphical model depicted in {numref}`graph_model_AttnLNPs_text`.
The latent path is implemented with the same method as LNPs, i.e. a mean aggregation followed by a parametrization of a Gaussian.
In other words, even though the deterministic representation is $R^{(t)}$ is target specific, the latent representation $\mathrm{Z}$  is target independent as seen in the computational graph ({numref}`computational_graph_AttnLNPs_text`).


```{figure} ../gifs/AttnLNP_single_gp.gif
---
width: 35em
name: AttnLNP_single_gp_text
alt: AttnLNP on single GP
---

Samples Posterior predictive of AttnLNPs (Blue) and the oracle GP (Green) with RBF,periodic, and noisy Matern kernel.
```

From {numref}`AttnLNP_single_gp_text` we see that although the marginal posterior predictive seem good the samples:
1. are not very smooth (the "kinks" seed in {numref}`AttnCNP_single_gp_text` are even more obvious when sampling);
2. lack diversity and seem to be shifted versions of each other. This is probably because having a very expressive deterministic path diminishes the need of a useful latent path.

Let us now look at images.

```{figure} ../gifs/AttnLNP_img.gif
---
width: 45em
name: AttnLNP_img_text
alt: AttnLNP on CelebA, MNIST, ZSMM
---
Samples from posterior predictive of an AttnCNP for CelebA $32\times32$, MNIST, ZSMM.
```

From {numref}`AttnLNP_img_text` we see that AttnLNP generates nice samples shows descent sampling and good performances when the model does not require generalization (CelebA $32\times32$, MNIST) but breaks for ZSMM as it still cannot extrapolate.

```{admonition} Details
---
class: tip
---
Model details, training and many more plots in {doc}`AttnLNP Notebook <../reproducibility/AttnLNP>`
```

## Convolutional Latent Neural Process (ConvLNP)

```{figure} ../images/graph_model_ConvLNPs.svg
---
width: 200em
name: graph_model_ConvLNPs_text
alt: graphical model ConvLNP
---
Graphical model for ConvLNPs.
```
```{figure} ../images/computational_graph_ConvLNPs.svg
---
width: 400em
name: computational_graph_ConvLNPs_text
alt: Computational graph ConvLNP
---
Computational graph for ConvLNPS. [simplify ? useful to mention or show induced points ? drop?]
```

The Convolutional LNPs {cite}`foong2020convnp` is the latent counterpart of ConvCNPs. A major difference compared to AttnLNP is that the latent path *replaces* the deterministic, which is done by actually having a latent functional representation (a latent stochastic process) instead of a latent vector valued variable. [intuition ...]

```{note}
An other way of viewing the ConvLNP is that it consists of 2 stacked ConvCNP, the first one models the latent stochastic process. The second one takes as input a sample from the latent stochastic process and models the posterior predictive conditioned on that sample.
```

* Mention that better trained using MLE probably because of the functional KL [better explanation ?]
* Link to theory
* talk about global representation ?


```{figure} ../gifs/ConvLNP_single_gp_extrap.gif
---
width: 35em
name: ConvLNP_single_gp_extrap_text
alt: ConvLNP on GPs with RBF, periodic, Matern kernel
---

Samples Posterior predictive of AttnLNPs (Blue) and the oracle GP (Green) with RBF,periodic, and noisy Matern kernel.
```

From {numref}`ConvLNP_single_gp_extrap_text` we see that ConvLNP performs very well and the samples are reminiscent of those from a GP, i.e., with much richer variability compared to {numref}`AttnLNP_single_gp_text`.

Let us now make the problem harder by having the ConvLNP model a stochastic process whose posterior predictive is non Gaussian. We will do so by having the following underlying generative process: sample one of the 3 previous kernels then sample funvtion. Note that the data generating process is not a GP (when marginalizing over kernel hyperparameters). Theoretically this could still be modeled by a LNPF as the latent variables could model the current kernel hyperparameter. This is where the use of a global representation makes sense.

```{figure} ../gifs/ConvLNP_kernel_gp.gif
---
width: 35em
name: ConvLNP_kernel_gp_text
alt: ConvLNP trained on GPs with RBF,Matern,periodic kernel
---
Similar to {numref}`ConvLNP_single_gp_extrap_text` but the training was performed on all data simultaneously.
```

From {numref}`ConvLNP_kernel_gp_text` we see that ConvLNP performs quite well in this harder setting. Indeed, it seems to model process using the periodic kernel when the number of context points is small but quickly (around 10 context points) recovers the correct underlying kernel. Note that we plot the posterior predictive of the actual underlying GP but the generating process is highly non Gaussian.

[should we also add the results of {numref}`ConvLNP_vary_gp` to show that not great when large/ uncountable number of kernels?]

```{figure} ../images/ConvLNP_marginal.png
---
width: 20em
name: ConvLNP_marginal_text
alt: Samples from ConvLNP on MNIST and posterior of different pixels
---
Samples form the posterior predictive of ConvCNPs on MNIST (left) and posterior predictive of some pixels (right).
```

As we discussed in  {ref}`the "CNPG" issue section <issues-cnpfs>`, CNP not only could not be used to generate coherent samples but the posterior predictive is also Gaussian.
{numref}`ConvLNP_marginal_text` shows that both of these issues are somewhat alleviated (compare to {numref}`ConvCNP_marginal_text`) [...]


Here are more image samples.

```{figure} ../gifs/ConvLNP_img.gif
---
width: 45em
name: ConvLNP_img_text
alt: ConvLNP on CelebA, MNIST, ZSMM
---
Samples from posterior predictive of an ConvCNP for CelebA $32\times32$, MNIST, ZSMM.
```

[REVERT PLOTS OF CONVLNP, I made a small modificiation which looks bad...]

### Issues of LNPFs

* Do not think that LNPF are necessarily better than CNP
* Cannot easily arg maximize posterior
* More variance when training
* More computationaly demanding to estimate (marginal) posterior predictive

[^LNPs]: In the literature the latent neural processes are just called neural processes. I use "latent" to distinguish them with the neural process family as a whole.
