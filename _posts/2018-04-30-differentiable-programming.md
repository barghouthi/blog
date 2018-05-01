---
title: "Differentiable Programming: A Semantics Perspective"
layout: post
---


So deep learning has taken the world by storm.
Frameworks for training deep neural networks, like [TensorFlow](https://www.tensorflow.org/), allow you to construct so-called *differentiable programs*.
The idea is that one can compute the derivative of some program, and then use that to optimize its parameters.
This post is written to indroduce researchers in the verification and programming languages community to automatic differentation of programs.
The assumption is that---extrapolating from myself here---you got into this feild for the love of logic and discrete math (and an unhealthy aversion to continuous mathematics).

---

## Programming language
Let's consider a very simple programming language where there are no loops or conditions, just a sequence of assignment statements the form:

$$v_1 \gets c \times v_2$$

$$v_1 \gets v_1 + v_2$$

$$v_1 \gets sin(v_2)$$

Here, $c$ is a real-valued constant and $v_i$ are real-valued program variables.
Any program $P$ in this language is assumed to have a single special input variable $x$ and an output variable $y$.

$P$ is also assumed to be in *static single assignment* (SSA) form---in other words, each variable gets assigned to once.
This is equivalent to *continuation-passing style* (CPS). If you've used TensorFlow, the *computation graph* that you construct there is effectively a program in SSA, where each graph node represents one variable's assignment.

## Langauge semantics
The semantics of our little language is standard.
A state $s$ of a program $P$ is a map from variables
to values.
The function $\textit{post}$ below takes a program and a state $s$ and returns the state resulting from executing $P$:

1. $\textit{post}(P_1;P_2, s) \triangleq \textit{post}(P_2,\textit{post}(P_1,s))$
2. $\textit{post}(v_1 \gets c \times v_2, s) \triangleq s[v \mapsto c \times s(v_2)]$
3. $\textit{post}(v_1 \gets v_2 + v_3, s) \triangleq s[v \mapsto s(v_1) + s(v_2)]$
4. $\textit{post}(v_1 \gets sin(v_2), s) \triangleq s[v \mapsto sin(s(v_2))]$

Above, $P_1;P_2$ denotes sequential composition.
The notation $s[v \mapsto c]$ denotes state $s$ but with $v$ mapping to the value $c$.




## Forward differentiation

Note that programs in our language are functions in $\mathbb{R} \to \mathbb{R}$.
We will now extend the semantics such that evaluating $P$ on input $x$ not only returns $P(x)$, but also $\frac{\partial P}{\partial x}(x)$, the partial derivative of $P$ w.r.t. the input variable $x$.
If you remember your calculus, the partial derivative is essentially the rate of change of the output $y$ as $x$ changes.

Below, we define the new semantics with a function $\partial\textit{post}$, where we keep track of two copies of the program variables, the variables $v_i$ and a new copy $\dot v_i$, which denotes the rate of change of $v_i$ w.r.t. the input $x$, i.e.,

$$\dot v_i = \frac {\partial v_i}{\partial x}(x)$$

Finally, when the program terminates with the new semantics, we can recover the variable $\dot y$, which will hold the value $\frac{\partial P}{\partial x}(x)$.

**Sequential composition** For sequential composition, $P_1;P_2$, $\partial\textit{post}$ behaves just like $\textit{post}$.

**Multiplication** For multiplication by a constant,

$$\partial\textit{post}(v_1 \gets c \times v_2) \triangleq s[v \mapsto c \times s(v_2)][\dot v \mapsto c \times \dot v_2]$$

In other words, the rate of change of $v_1$ w.r.t. $x$ is the rate of change of $v_2$ scaled by $c$.

**Addition** For addition, we have

$$\partial\textit{post}(v_1 \gets v_2 + v_3) \triangleq s[v \mapsto s(v_1) + s(v_2)][ \dot v \mapsto s(\dot v_1) + s(\dot v_2)]$$

That is, the rate of change $v_1$ is the sum of the rates of change of $v_2$ and $v_3$.

**Trignometric functions** For sine, we have

$$\partial\textit{post}(v_1 \gets sin(v_2)) \triangleq s[v \mapsto sin(s(v_2))] [\dot v \mapsto \dot v_2 \times cos(s(v_2))]$$

This follows from the *chain rule*, which says that the rate of change of $f(u)$ is the rate of change of $f$ scaled by the rate of change of its argument $u$.
You might remember that derivative of $sin(x)$ is $cos(x)$, so, following the chain rule, we simply scale $cos(v_2)$ by $\dot v_2$.

## Notes

I covered the simpler case of forward differentiation, which proceeds by executing the program in a forward manner. For functions with more than one input, it is more efficient to perform backward differentiation, which the popular *backpropagation* algorithm implements. Adapting the above semantics to backpropagation is not hard, it's just messier, as we have to execute the program forward and then backward. Therefore, I decided to illustrate the forward mode only. For more information, I encourage you to read the excellent survey by [Baydin et al.](https://arxiv.org/abs/1502.05767), which heavily influenced my presentation.
