---
title: "Teaching Your SMT Solver Probability Theory"
layout: post
---

The unexpected rise of SAT and SMT solvers has revolutionized software verification, both the automated and deductive flavors.
Simply, you encode program semantics as logical circuits and ask the SMT solver questions about them. How elegant.
But what happens when your program is randomized? Good luck! The first-order world of SMT solvers does not have the ingredients to sustain your stochastic existence. Go find another home.

In this post, I will show you how to teach your SMT solver probability theory.
The ideas here are a simplified view of a recent [POPL paper](http://pages.cs.wisc.edu/~aws/papers/popl19.pdf) by my student [Calvin Smith](http://pages.cs.wisc.edu/~cjsmith/).

---

## Classical verification

To ground things, I will start with a classical (non-probabilistic) verification problem and encode it as a formula first-order.
Take this program:

```python
def f(x):
    z = x + 1
    y = z * 2
    return y
```

If we're working with infinite-precision integers, the following Hoare triple is valid---a positive input results in a positive output.

$$\vdash \{x > 0\}  ~f(x)~ \{y > 0\}$$

To formally prove that this Hoare triple holds, we encode the precondition, postcondition, and program semantics as the following formula:

$$(\underbrace{x > 0}_{\text{pre}} \land \underbrace{z = x + 1 \land y = z * 2}_{\text{encoding of } f}) \Longrightarrow \underbrace{y > 0}_{\text{post}}$$

If the SMT solver tells you that the formula is valid, then the Hoare triple is valid. If it's not valid, the SMT solver will give you a counterexample.

## Randomized algorithm example

Let's now look at a very simple randomized program, where `uniform(a,b)` returns a sample from the uniform distribution between values `a` and `b`. 

```python
def f(x):
    y = uniform(0,3*x)
    return y
```

Say we want to prove the following (probabilistic) Hoare triple:

$$\vdash_{\color{red}{1/3}} \{x > 0\}  ~f(x)~ \{y \geq x\}$$

Let's unpack this: If $x$ is positive, then $f$ returns a value of $y \geq x$, *but* there is at most a $\color{red}{1/3}$ probability of failure.

This Hoare triple is intuitively valid: values of $y$ are uniformly distributed between $0$ and $3x$, so getting a value of $y \geq x$ has a failure probability of $1/3$.

Cool. But we want to automatically establish this Hoare triple with an SMT solver.
How? We'll get rid of probability. Adios!

## Turning probability to non-determinism

The idea is that the SMT solver needs to only know a few *axioms* about the probability distributions in order to construct the proof.
In our example, the proof relies on the obvious fact that $y \geq x$ with a probability of $2/3$.
If we know this fact, we can transform the program into a non-deterministic version that *tracks probability of failure*:

```python
def f_new(x):
    y = pick a value in [x,...,3*x]
    w = 1/3
    return y,w
```

We now have a non-probabilistic program:
`y`  gets an arbitrary (non-deterministic) value between `x` and `3*x`;
but we know that this may not be true with a probability of `1/3`, which is stored in a new *ghost* variable `w`.
The transformation relies on the following insight:

1. make whatever assumptions you want
2. *but* remember the probability with which your assumptions might fail

So now we can prove the above Hoare triple $$\vdash_{\color{red}{1/3}} \{x > 0\}  ~f(x)~ \{y \geq x\}$$  using the transformed, non-deterministic program instead:

$$(\underbrace{x > 0}_{\text{pre}} \land \underbrace{x \leq y \leq 3x  \land w = 1/3}_{\text{encoding of } f_\textit{new}}) \Longrightarrow (\underbrace{y > x}_{\text{post}} \land \underbrace{w \leq \color{red}{1/3}}_{\text{failure prob.}})$$


