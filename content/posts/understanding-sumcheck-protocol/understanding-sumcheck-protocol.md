---
title: "Breaking down the Sum-Check Protocol"
date: 2024-06-06T08:17:00+02:00
draft: false
---

Few months ago I implemented [my own version of the sum-check protocol](https://github.com/MatteoMer/rust-sumcheck), and it was pretty fun to do!

At the time I started writing an article that never got published, so today I'm finally taking some time to finish it

I assume you know some Rust, finite fields and polynomials

## Getting started
The goal of the protocol is for a prover to convince a verifier that a certain number $H$ is the sum of all the evaluations of a polynomial $g(x_1,...,x_n)$

{{<notice note>}}
In this article, we're going to use the evaluations _over the boolean hypercube_, meaning the variables will only be $0$ or $1$.
{{</notice>}}

**Example:**
With a polynomial $g(x_0,x_1,x_2)=2{x_0}^3+x_1+x_0x_2$, a prover can easily compute:

$H := g(0,0,0)+g(0,0,1)+g(0,1,1)+g(1,1,1)+g(1,1,0)+g(1,0,0)+g(1,0,1)+g(0,1,0)$

$H := 0+0+1+4+3+2+3+1$

$H := 14$

The goal here for the prover is to convince the verifier that $H$ has been computed correctly without the need of the verifier to compute it himself!

The first thing we can do is to define our polynomial. I'm using arkworks here for the polynomial and field artithmetics

```rust
main.rs

fn main() {
  
  // g(x_0, x_1, x_2) = 2*{x_0}^3 + x_1 + x_0*x_2
  let g = SparsePolynomial::from_coefficients_vec(
    3, 
    vec![
      (Fq::from(2), SparseTerm::new(vec![(0, 3)])), 
      (Fq::from(1), SparseTerm::new(vec![(2, 1)])),
      (Fq::from(1), SparseTerm::new(vec![(1, 1), (2, 1)])),
    ],
  );

  let prover = Prover::new(&g).unwrap();

}
```

Now let's create a Prover that computes the sum when creating a new instance

```rust
prover.rs

impl<F> Prover<F, SparsePolynomial<F, SparseTerm>>
where
    F: Field + From<i32>,
{
    fn sum_evaluation(g: &SparsePolynomial<F, SparseTerm>) -> Option<F> {
        let v = g.num_vars();
        let mut sum = F::zero();

        // all {0,1}^3 combination
        for i in 0..(1 << v) {
            let point: Vec<F> = (0..v)
                .map(|d| {
                    if (i >> d) & 1 == 1 {
                        F::from(1)
                    } else {
                        F::from(0)
                    }
                })
                .collect();

            // Evaluates the polynomial at the generated point, summing the results.
            sum = sum.add(&g.evaluate(&point));
        }
        Some(sum)
    }
    pub fn new(g: &SparsePolynomial<F, SparseTerm>) -> Option<Self> {
        Some(Self {
            g: g.clone(),
            h: Prover::sum_evaluation(g)?,
        })
    }
```

## Just prove it
The first step of the protocol is for the prover to send a polynomial $g_0$ to the verifier. 
It's an univariate polynomial defined like this: $$g_0(x_0)=g(x_0,0,0)+g(x_0,1,0)+g(x_0,0,1)+g(x_0,1,1)$$

You can see it's very similar to the initial sum but restricted to varying only the variable $x_0$ while other variables $x_1$ and $x_2$ are summed over all possible values (0 or 1).

This polynomial is useful, because if the verifier is sure that the polynomial is well formed, he can compute $H$ by himself doing: $$H := g_0(0)+g_0(1)$$

How can the verifier be sure that $g_0$ is well formed? For that, we can leverage the [Schwartz-Zippel Lemma](https://publish.obsidian.md/matteo/3.+Permanent+notes/Schwartz-Zippel+Lemma)

The verifier will sample a random $r_0\in\mathbb{F}$ and he only needs to check if $$g_0(r_0)=g(r_0,0,0)+g(r_0,1,0)+g(r_0,0,1)+g(r_0,1,1)$$If this is valid, then the polynomial is correctly formed

We still have a problem, since it's still very expensive for the verifer to compute the RHS (the entire sum needs $2^3$ evaluations to computed and the RHS $2^2$, so only a factor of two less).

To solve this problem, let's first note that $g_0(r_0)$ is actually a value, we can call it $H_1$ for example. So now we have: $$H_1=g(r_0,0,0)+g(r_0,1,0)+g(r_0,0,1)+g(r_0,1,1)$$ Maybe you already realized, but this actually the perfect setting to call the sum-check protocol! The idea is to basically applying the protocol recursively until the last step, where the verifier can do one call to $g$ with only random values, making sure all previous steps are correct!

#### Example
Let's take our initial sum-check instance we want to prove: $H=14$ with $$g(x_0,x_1,x_2)=2{x_0}^3+x_1+x_0x_2$$

1. Prover sends the verifier $g_0=8{x^3}+2x+2$
2. Verifier makes sure that $g_0(0)+g_0(1)=H$ then samples a random $r_0=12$ and compute $g_0(12)=13850$
3. Prover summon a new sum-check instance where he wants to prove that $13850=g(12,0,0)+g(12,0,1)+g(12,1,0)+g(12,1,1)$
4. Prover sends $g_1=6924+2x$
5. Verifier makes sure that $g_1(0)+g_1(1)=g_0(r_0)$ then samples a random $r_1=5$ and compute $g_1(r_1)=6934$
6. Prover summon a new sum-check instance where he wants to prove that $6934=g(12,5,0)+g(12,5,1)$
7. Prover sends $g_2=3461+12x$
8. Verifier makes sure that $g_2(0)+g_2(1)=g_1(r_1)$ then samples a random $r_2=2$ and execute the final check where he evaluates $g$ and verify that $g_2(r_2)=g(12,5,2)$, if this is true, the sum-check protocol is valid!

{{<notice note>}}
At each step, the verifier also checks that $deg(g_i)\le{deg(g)}$, to make sure the prover is not cheating.
{{</notice>}}

Let's get back to our code! Let's start by the first round (I decided to code the iterative version of the protocol, not recursive, because it's easier!)
```rust
interactive.rs

pub fn interactive_protocol<F>(
    p: &Prover<F, SparsePolynomial<F, SparseTerm>>,
    v: &Verifier<F, SparsePolynomial<F, SparseTerm>>,
) -> bool
where
    F: Field + From<i32>,
{
    let mut r: Vec<F> = Vec::new();

    // first round
    let g_1 = Prover::construct_uni_poly(&p.g, &r, 0);
    if !v.check_claim(&g_1, p.h, 0) {
        panic!("[sumcheck-interactive] claimed failed at first round");
    }
    let mut r_i = v.send_random_challenge();
    r.push(r_i);

    let mut c_i = g_1.evaluate(&r_i);

    //TODO: next rounds
}
```

Few points to note:
- `let mut r: Vec<F> = Vec::new();` will contain all random values that the verifier sent, we need to keep track of them to be able to evaluate $g$ at these points.
- We need to add to our prover a function `fn construct_uni_poly(g: &Polynomial, r: &[F], round: usize);` 
- We need to create a verifier and write a function `fn check_claim(g_i: &DensePolynomial, c_i, round)` that verifies that the claim is valid
- `v.send_random_challenge()` is trivial to write
- `g_1.evaluate(&r_i)` is already defined in `DensePolynomial` in arkworks 

Let's start by constructing our univariate polynomial, I adapted the great work of [punwai](https://github.com/punwai/sumcheck/blob/main/src/main.rs) for that, I remember being a bit stuck on that for some reasons, but in the end, I adapted his code and it works great!
```rust
prover.rs

// attribution: https://github.com/punwai/sumcheck/blob/main/src/main.rs
    pub fn construct_uni_poly(
        g: &SparsePolynomial<F, SparseTerm>,
        r_i: &[F],
        round: usize,
    ) -> DensePolynomial<F> {
        let mut coefficients = vec![F::zero(); g.degree() + 1];
        let v = g.num_vars();

        // number of inputs to generate, we substract round because it's the nb of already known
        // inputs at the round; at round 1 we will have r_i.len() = 1
        for i in 0..2i32.pow((v - round - 1) as u32) {
            let mut inputs: Vec<F> = vec![];
            // adding inputs from previous rounds
            inputs.extend(r_i);
            // adding round variable
            inputs.push(F::zero());
            // generating inputs for the rest of the variables
            let mut counter = i;
            for _ in 0..(v - round - 1) {
                if counter % 2 == 0 {
                    inputs.push(0.into());
                } else {
                    inputs.push(1.into());
                }
                counter /= 2;
            }

            //computing polynomial coef from evaluation
            for (c, t) in g.terms.clone().into_iter() {
                let mut c_acc = F::one();
                let mut which = 0;

                for (&var, pow) in t.vars().iter().zip(t.powers()) {
                    if var == round {
                        which = pow;
                    } else {
                        c_acc *= inputs[var].pow([pow as u64]);
                    }
                }
                coefficients[which] += c * c_acc;
            }
        }

        DensePolynomial::from_coefficients_vec(coefficients)
    }

```

Now let's implement our verifier!
```rust

impl<F, P> Verifier<F, P>
where
    F: Field + From<i32>,
    P: DenseMVPolynomial<F>,
{
    pub fn send_random_challenge(&self) -> F {
        let mut rng = rand::thread_rng();

        let r: u8 = rng.gen();
        F::from(r)
    }

    pub fn check_claim(&self, g_j: &DensePolynomial<F>, c_j: F, round: usize) -> bool {
        // check if g_j(0) + g_j(1) = c_j
        let eval_zero = g_j.evaluate(&F::zero());
        let eval_one = g_j.evaluate(&F::one());
        if eval_zero + eval_one != c_j {
            return false;
        }

        // check if deg(g_j) <= deg_j(g)
        let deg_g = get_variable_degree(&self.g, round);
        let deg_g_j = g_j.degree();
        if deg_g_j > deg_g {
            return false;
        }

        true
    }


```

Nothing super fancy here, the verifier job is quite easy! (As expected!)

Now the rest of the code is very similar, you can check it [here](https://github.com/MatteoMer/rust-sumcheck/blob/d8453c27b88448115a863943776cdcba86363f5b/src/interactive.rs#L36) (all rounds are doing the same thing)

I also implemented a non-interactive version [here](https://github.com/MatteoMer/rust-sumcheck/blob/main/src/non_interactive.rs), you could do it as well pretty easily using [merlin](https://github.com/dalek-cryptography/merlin/tree/master/src) or [nimue](https://github.com/arkworks-rs/nimue)

Hope you enjoyed this cool article, and understood a bit more the sum-check protocol! 

I also recommend:
- The [video of David Wong](https://www.youtube.com/watch?v=XV62OB022tU), where he explain the sum-check protocol in 5 minutes (!).
- [Proofs Arguments and Zero-Knowledge](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf), chapter 4.1, which is where I learned about the protocol the first time!

You can also check my [obsidian notes about the sum-check protocol](https://publish.obsidian.md/matteo/3.+Permanent+notes/Sum-Check+Protocol). They are using more mathematical notations (and with more details!).

I'm going to try write more articles like this in the future, it's fun and forces me to really understand deeply the protocol I'm learning (both by writing the code and the article). Right now I'm working on GKR, I hope to be able to have a working version soon, you can check my progress [here](github.com/matteomer/rust-gkr)
