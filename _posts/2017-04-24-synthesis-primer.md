---
title: "A Program Synthesis Primer"
layout: post
---

My colleague Somesh Jha recently asked me to give a lecture on *program synthesis* to his class.
As I prepared my notes, I realized that a single lecture is long enough to formally define the problem and code up some cool examples
that demonstrate the process.
This post contains the notes and code I used in class.{% sidenote 'code'
'I go through two simple examples in this post; the full code is on [GitHub](https://github.com/barghouthi/704examples/tree/master/synthesis).' %}

## What is program synthesis anyways?

In the fields of programming languages and verification,
the traditional problem of program synthesis involves
constructing a program from a high-level mathematical specification.
For example, suppose you want to write a program
that computes the factorial of a number {% m %} n \gt 1 {% em %}.
You start with the mathematical definition of factorial,
{% m %} n! {% em %}, and you gradually massage
it---rewrite it using your knowledge of the properties of
factorial and mathematics---until you get to an executable program.

While lofty in its goals and rich in its tradition, this synthesis approach---called *deductive synthesis*---has had limited success.
Simply, mathematical specifications are hard to write,
and turning a specification to a program is very hard to automate.

## Combinatorial synthesis

Recently, researchers have been looking at forms
of synthesis that are simpler to automate.
The style of synthesis I will focus on here is
the one pioneered in the [Sketch](https://people.csail.mit.edu/asolar/papers/thesis.pdf) project.
The idea is that we start with a *reference implementation* ```p_naive```.
Think of this as the naive solution of a problem
that you're sure works correctly, but that you won't put into production, perhaps because it's pretty inefficient.
Your goal is to find a program ```p_smart```
that is equivalent to ```p_naive```,
but that is more efficient and hard to get right.{% sidenote 'compiler' '"But this is what an optimizing compiler does!" you protest. Yes! But here we are taking a *combinatorial* approach: a compiler applies a fixed set of tricks to get to an efficient program; here we will manually define a space of programs, and search it exhaustively until we find the right one. In some sense, we are doing *superoptimization*'%}


For illustration, let's look at an extremely simple example.
The naive program is the following,
where for simplicity we assume all ```ints``` are bitvectors with an 8-bit width.

``` c
int p_naive(int x):
    y = x * 2
    return y
```
You, as a smart (but not very smart) programmer, think
that you can do better by writing a program of the following
form, but you're not sure what to place in the ```??```

``` c
int p_smart(int x):
    y = x << ?? // the operator << is shift left
    return y
```
In a sense, ```p_smart``` represents a family
of programs---all possible instantations of the *hole* ```??```.
Of course, the right completion here is replacing
the hole with ```1```.

## Automating synthesis

So how do we automatically find the value of ```??```
that makes ```p_smart``` equivalent to ```p_naive```.
Easy! There are {% m %} 2^8 {% em %} different instantiations 
of ```??```, so we enumerate
them until we find the right one.
In general this incurs a combinatorial explosion.
We cannot avoid the combinatorial problem,
but it turns out we can neatly characterize the
search space as a formula that we can give to an efficient
off-the-shelf solver---as we will see in a bit.

First, let's try to gradually formalize the problem.
We will use the variable ````h````
to denote the hole ```??```.

**Definition 1** *Find a value* for ```h```
such that *for all values* of ```x```,
```p_naive(x) == p_smart(x)```.

**Definition 2**  We can think of ```h``` as
an input to ```p_smart```.
So, our goal is really to
*find a value* for ```h```
such that *for all values* of ```x```
```p_naive(x) == p_smart(x,h)```.

**Definition 3** Now, we can view a program, say, ```p_naive```,
as a logical relation {% m %} \varphi_n(x,y) {% em %}
over its input and output variables, {%m%} x {%em%} and {%m%}y{%em%}.
The idea is that the relation is true if and only
if ```p_naive(x) == y```.

So, our goal is really to find
*find a value* for {%m%} h {%em%}
such that *for all values* of  {%m%} x {%em%} and {%m%}y{%em%},
we have
{% m %} \varphi_n(x,y) \iff \varphi_s(x,h,y) {% em %}
is true.

**Final definition**
Finally, we are ready to state the problem logically.
Find a satisfying assignment of the following formula
{%math%}
\forall x,y \ldotp \varphi_n(x,y) \iff \varphi_s(x,h,y)
{%endmath%}
Notice that the variable {%m%} h {%em%}
is *free*. Therefore, any satisfying assignment
of this formula is only over {%m%} h {%em%}.

At this point, we have completely
reduced the synthesis problem to solving a logical
formula. We can do this with an SMT solver like [Z3](https://github.com/Z3Prover/z3).

## A detailed example

We can now apply the above
process to encode ```p_smart``` and ```p_naive``` as formulas.
We will encode them in the first-order theory of bitvectors,
which has all the standard operations we care about.

{% math %}
\varphi_n(x,y) \equiv y = x * 2
{% endmath %}

{% math %}
\varphi_s(x,y,h) \equiv y = x << h
{% endmath %}

We will now encode those in Z3
using its Python API
as follows.

First, we define all the variables
as 8-bitvectors.

*The code for this example is available
on [GitHub](http://github.com/barghouthi/704examples/blob/master/synthesis/synth.py)*

```python
x = BitVec('x',8)
y = BitVec('y',8)
h1 = BitVec('h',8)
```

Next, we encode {%m%} \varphi_n {%em%} and {%m%} \varphi_s {%em%}:
```python
phi_n = y == x * 2
phi_s = y == x << h
```

Finally, we are ready to invoke Z3:

```python
# first, encode the universally quantified formula
# (the == symbol is if and only if)
encoding = ForAll([x,y], phi_n == phi_s)
# second, call Z3 and check if there is a model
s = Solver()
s.add(encoding)
s.check()
# print the model (i.e., the value of h)
print s.model()
```

If you run this code, you will get the output ```[h = 1]```,
just as expected!

## Test-driven synthesis

The approach presented above works over two programs:
it tries to find a program that is *equivalent* to
some reference implementation.
Instead of writing a full-blown reference implementation,
we will now just write some test cases,
and find a program that passes all of them.
You could think of this as test-driven development on steroids!

Say we want to write a function that, given
bitvector ```x```, returns a bitvector ```y``` that is 1  in the
position of the first 0 from the right occuring in ```x```, and 0 everywhere else.{% sidenote 'sketch_example' 'Example borrowed from Sketch project and found through Loris D’Antoni’s notes.' %}
For example,
``` python
# test 1
x = 00000000
y = 00000001
# test 2
x = 00000011
y = 00000100
# test 3
x = 00000010
y = 00000001
```

Suppose you have the following hunch on how to write
such a program efficiently:
```c
y = (~(x + ??)) & (x + ??)
```
Here ```~``` is bitwise not and ```&``` is bitwise and.
Our goal is to find solutions to the two holes
that result in our desired program.

Just as before, we encode the program as a logical relation:
{% math %}
\varphi_s \equiv y = (\sim(x + h_1))\ \&\ (x + h_2)
{% endmath %}
Note how we encoded the two holes as two variables.

Similarly, we can encode the test cases as a relation:
{% math %}
\varphi_t \equiv  t_1 \lor t_2 \lor t_3
{% endmath %}
where
{% math %}
t_1 \equiv x = 0 \land y = 1\\
t_2 \equiv x = 3 \land y = 4\\
t_3 \equiv x = 2 \land y = 1
{% endmath %}

Finally, we solve the following formula:
{%math %}
\forall x, y \ldotp \varphi_t \Rightarrow \varphi_s
{% endmath %}
Observe that we have an implication and not an equivalence
between the two relations.
Intuitively, the relation defined by the test cases
is small and does not define the behavior other than for
the 3 test cases we've supplied.
So what we're looking for is a program ```p_smart```
where all
test cases appear in (or are a subset of) its relation.

Let's encode this in Z3.
The process is similar to what we saw above:

*The code for this example is available
on [GitHub](http://github.com/barghouthi/704examples/blob/master/synthesis/synth_test.py)*

```python
phi_s = y == (~(x + h1)) & (x + h2)
t1 = And(x == 0, y == 1)
t2 = And(x == 1, y == 2)
t3 = And(x == 2, y == 1)
phi_t = Or(t1,t2,t3)

encoding = ForAll([x,y], Implies(phi_t, phi_s))
```

Solving the formula ```encoding```,
we get ```[h2 = 1, h1 = 0]```, which results in a correct program.

Since we are dealing with test cases,
the test cases we supply may be insufficient to force the SMT
solver to find the right program; in such case,
you can supply more tests.
For example, if I just give Z3 test 3, on my machine I get
```[h2 = 15, h1 = 14]```, which is incorrect (check it).

## Conclusion
We looked at a combinatorial form of program synthesis, where we define the search space as a program with holes.
You might wonder how to encode control-flow, and not only holes that
are replaced with consants. A great paper to read is [Synthesis of Loop-free Programs](http://www.csl.sri.com/users/tiwari/papers/pldi2011-bitvector.pdf).

The idea of searching the space of programs has recently
been explored in a wide array of settings. It turns out
that handing the problem to the SMT solver is not always the right
way to go; sometimes a custom enumeration algorithm outperforms
SMT solvers. SMT solvers, however, are very good at finding
*magic constants*, which simple enumeration may never get to.
So, in bit-twiddling problems, symbolic encodings like the one shown here are the way to go.
