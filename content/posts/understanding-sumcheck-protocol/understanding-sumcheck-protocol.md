---
title: "Understanding the Sum-check Protocol"
date: 2024-04-15T08:17:00+02:00
draft: true
---

[Jolt Alpha](https://a16zcrypto.com/posts/article/accelerating-the-world-computer-implementing-jolt/) has been released recently, and by a fun coincidence, I was studying the sum-check protocol at this time.

So I thought that an article introducing this very useful protocol, alongside a Rust implementation could help some people understanding better what primitives Jolt is using!

In this article, I assume you know some Rust, finite field arithmetics and polynomials! 

# Purpose of the protocol
The purpose of the sum-check protocol can be defined in a single sentence: a prover $\mathcal{P}$ wants to convince to a verifier $\mathcal{V}$ that he knows the sum of all the evaluations of a polynomial over all Boolean inputs.

Let's try to understand it, step-by-step:
#### A prover $\mathcal{P}$ wants to convince to a verifier $\mathcal{V}$ that he knows a claim
The idea here is that we have a prover (named $\mathcal{P}$) that knows something (a claim $y$ for example), and he wants to convince a verifier (named $\mathcal{V}$) that this claim is valid.

But $\mathcal{V}$ is lazy, and doesn't want to do the calculation himself to verify the claim $y$. That's where our protocol is useful: $\mathcal{V}$ can be convinced that $\mathcal{P}$ isn't lying without doing the entire computation himself.
#### The sum of all the evaluations of a polynomial over all Boolean inputs
Here, the claim of $\mathcal{P}$ is, for a polynomial $g$, he computed every possible combination of inputs that are either 0 or 1 (the Boolean inputs), and then summed all the combination to get a number $H$.

Let's take an example with a polynomial $g(x,y)=2x+y$. Here $H$ would be: 
$$
H=g(0,0)+g(0,1)+g(1,0)+g(1,1)=0+1+2+3=6
$$

So we can summarize the protocol by saying that a prover $\mathcal{P}$ wants to convince a verifier $\mathcal{V}$ that he knows $H$ for a polynomial $g$

{{<notice note>}}
We can use any set of inputs, not the boolean inputs: but in this article we will stick to the boolean inputs
{{< /notice>}}

# A quick, but important detour
In this protocol, we make great use of the [Schwartz-Zippel Lemma](https://publish.obsidian.md/matteo/3.+Permanent+notes/Schwartz-Zippel+Lemma).

This lemma is basically saying that for two polynomials $p$, $q$ of degree $d$ and a randomly chosen $x\in\mathbb{F}$, the probability that $p(x)=q(x)$ is at most $d/|\mathbb{F}|$.

The idea is that we will leverage this Lemma to check if two polynomials are equal. It will make more sense during the protocol, but it's important to understand that we will be doing some probabilistic checks!


# Intuition behind the protocol
Before going into the Rust implementation, let's try to build some intuition on the protocol; the implementation will be slightly different, but it will help understanding how it works!

In the protocol, we use $g$, a $v$-variate polynomial defined over a finite field $\mathbb{F}$. It means that our polynomial $g$ has $v$ variable; for example for $g(x,y,z)$, $v=3$.

We also define $H$ like this: 
$$
H: \sum_{{b_1}\in\\{0,1\\}} \sum_{{b_2}\in\\{0,1\\}} \cdots \sum_{{b_v}\in\\{0,1\\}} g(b_1,b_2,...,b_v) 
$$
 (it's the mathematical way of writing $H$ as we defined it just before)


### The recursive-version of the Protocol
Our protocol starts with $\mathcal{P}$ sending a value $C_1$ claimed to be equal to the value $H$ to $\mathcal{V}$.

How can the verifier know if $H$ is valid? He could compute $H$ himself, but the goal of this protocol is avoid that.

Instead, he will ask the prover to send an univariate polynomial $g_1$ and compute $$C_1 \stackrel{?}{=} g_1(0)+g_1(1)$$

If the result is true, and if we are sure that $g_1$ has been correctly formed, by an honest prover, we can stop the protocol here!

Here, we have to trust the prover on forming the polynomial correcly, which is not ideal at all. 
What the verifier could do to avoid this, is constructing the polynomial (let's call it $s_1$) by himself, to make sure it's correctly formed!

Then using the Schwartz-Zippel Lemma described earlier, we can do a probabilistic check that $s_1(r_1)=g_1(r_1)$ for a random $r_1\in\mathbb{F}$.
Now $\mathcal{V}$ is sure that the polynomial is correctly formed, so $C_1$ is valid!

##### Evaluating $s_1(r_1)$
The verifier doesn't really want to construct $s_1$ himself, so instead he will take advantage on an interesting fact of $s_1$:

$s_1$ is the sum of the evaluation of a $(v-1)$-variates polynomial. It means that evaluating $s_1(r_1)$ is equal to: 
$$
s_1(r_1)=\sum_{(x_2,\cdots,x_v)\in\\{0,1\\}^{v-1}}g(r_1,x2,\cdots,x_v)
$$

What's interesting here is that we are trying get the **sum** of the evaluations of a **$(v-1)$-variate polynomial**.

Maybe you got it already, but this is exactly what the sum-check protocol is designed to check! 

##### Recursion time
We will apply recursively the sum-check protocol, until the last round where $s_v$ will be equal to 
$$
s_v=g(r_1,\cdots,r_v)
$$

At this last round, the verifier can construct $s_v$ by himself, compute $s_v(r_v)$, and verify that all the previous claims were valid.

### An example to understand better
I know, it must feel confusing. Let's go through an example so it's easier to understand.

Let's take $g(x,y,z)=2x^3 + xy + yz$.

$\mathcal{P}$ claims that $H=12$. You don't have to believe him, we will get convinced by the sum-check protocol!

##### First round
As we just said, $\mathcal{P}$ send $\mathcal{V}$ the value $C_1=12$. $\mathcal{P}$ also sends $g_1$ to $\mathcal{V}$ 
$$g_1(X_1)=8X^3_1+2X_1+1$$
The verifier checks if $g_1$ is valid: $$g_1(0)+g_1(1)=(0+0+1) + (8+2+1) = 12$$

