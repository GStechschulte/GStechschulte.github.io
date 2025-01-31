+++
title = 'Variational Inference - Evidence Lower Bound'
date = 2022-06-03T18:43:46+02:00
author = 'Gabriel Stechschulte'
categories = ['probabilistic-programming']
draft = false
math = true
+++

We don't know the real posterior so we are going to choose a distribution $Q(\theta)$ from a family of distributions $Q^*$ that are **easy to work with** and parameterized by $\theta$. The approximate distribution should be *as close as possible* to the true posterior. This closeness is measured using KL-Divergence. If we have the joint $p(x, z)$ where $x$ is some observed data, the goal is to perform inference: given what we have observed, what can we infer about the latent states?, i.e , we want the posterior.

Recall Bayes theorem:

$$p(z | x) = \frac{p(x|z)p(z)}{p(x)}$$

The problem is the marginal $p(x = D)$ as this could require a hundred, thousand, . . .dimensional integral:

$$p(x) = \int_{z_0},...,\int_{z_{D-1}}p(x, z)dz_0,...,d_{z_D{-1}}$$

If we want the full posterior and can't compute the marginal, then what's the solution? **Surrogate posterior**. We want to approximate the true posterior using some known distribution:

$$q(z) \approx p(z|X=D)$$

where $\approx$ can mean you want the approximated posterior to be "as good as possible". Using variational inference, the objective is to minimize the distance between the surrogate $q(z)$ and the true posterior $p(x)$ using KL-Divergence:

$$q^*(z) = argmin_{q(z) \in Q} (KL(q(z) || p(z|x=D)))$$

where $Q$ is a more "simple" distribution. We can restate the KL-divergence as the expectation:

$$KL(q(z) || p(z|D)) = \mathbb{E_{z \sim q(z)}}[log \frac{q(z)}{p(z|D)}]$$

which, taking the expectation over $z$, is equivalent to integration:

$$\int_{z_0}, . . .,\int_{z_{D-1}}q(z)log\frac{q(z)}{p(z|D)}d_{z_0},...,d_{z_{D-1}}$$

But, sadly we don't have $p(z \vert D)$ as this is the posterior! We only have the joint. Solution? Recall our KL-divergence:

$$KL(q(z) || p(z|D))$$

We can rearrange the terms inside the $log$ so that we can **actually** compute something:

$$\int_{z}q(z)log(\frac{q(z)p(D)}{p(z, D)})dz$$

We only have the **joint**. Not the posterior; nor the marginal. We know from Bayes rule that we can express the posterior in terms of the joint $p(z, D)$ divided by the marginal $p(x=D)$:

$$p(z|D) = \frac{p(Z, D)}{p(D)}$$

We plug this inside of the $log$:

$$\int_{z}q(z)log(\frac{q(z)p(D)}{p(z, D)})dz$$

However, the problem now is that we have reformulated our problem into **another** quantity that we don't have, i.e., the marginal $p(D)$. But we can put the quantity that we don't have outside of the $log$ to form two separate integrals.

$$\int_z q(z)log(\frac{q(z)}{p(z, D)})dz + \int_zq(z)log(p(D)dz$$

This is a valid _rearrangement_ because of the properties of logarithms. In this case, the numerator is a product, so this turns into a sum of the second integral. What do we see in these two terms? We see an expectation over the quantity $\frac{q(z)}{p(z, D)}$ and another expectation over $p(D)$. Rewriting in terms of expectation:

$$\mathbb{E_{z{\sim q(z)}}}[log(\frac{q(z)}{p(z, D)})] + \mathbb{E_{z \sim q(z)}}[log(p(D))]$$

The right term contains **information we know**—the functional form of the surrogate $q(z)$ and the joint $p(z, D)$ (in the form of a directed graphical model). We still don't have access to $p(D)$ on the right side, but this is a **constant quantity**. The expectation of a quantity that does not contain $z$ is just whatever the expectation was taken over. Because of this, we can again rearrange:

$$-\mathbb{E_{z \sim q(z)}}[log \frac{p(z, D)}{q(z)}]+log (p(D))$$

The minus sign is a result of the "swapping" of the numerator and denominator and is required to make it a _valid_ change. Looking at this, the left side is a function dependent on $q$. In shorthand form, we can call this $\mathcal{L(q)}$. Our KL-divergence is:

$$KL = \mathcal{-L(q)} + \underbrace{log(p(D))}_\textrm{evidence}$$

where $p(D)$ is a value between $[0, 1]$ and this value is called the **evidence** which is the **log probability of the data**. If we apply the $log$ to something between $[0, 1]$ then this value will be negative. This value is also **constant** since we have observed the dataset and thus does not change.

$KL$ is the **distance** (between the posterior and the surrogate) so it must be something positive. If the $KL$ is positive and the evidence is negative, then in order to fulfill this equation, $\mathcal{L}$ must also be negative (negative times a negative is a positive). The $\mathcal{L}$ should be *smaller* than the evidence, and thus it is called the **lower bound** of the evidence $\rightarrow$ **Evidence Lower Bound** (ELBO).

Again, ELBO is defined as: $\mathcal{L} = \mathbb{E_{z \sim q(z)}}[log(\frac{p(z, D)}{q(z)})]$ and is important to note that the ELBO is equal to the evidence if and only if the KL-divergence between the surrogate and the true posterior is $0$:

$$\mathcal{L(q)} = log(p(D)) \textrm{ i.f.f. } KL(q(z)||p(z|D))=0$$
