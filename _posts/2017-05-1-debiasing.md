---
title: "Fairification: Making Unfair Programs Fair"
layout: post
---

Over the past year, I have been exploring the notion of *algorithmic fairness* from a PL/verification perspective.
Today I'm going to talk about a new paper we have that is appearing at CAV 2017: [*Repairing Decision-making Programs under Uncertainty*](http://pages.cs.wisc.edu/~aws/papers/fatml16.pdf), with Samuel Drews and Loris D'Antoni.

## Algorithmic fairness

With software rapidly overtaking sensitive decision-making processes, like policing and sentencing, many people have been very concerned with unfairness in automated decision-making.
The past year or so has seen lots of attention in this space, for example:
* [ProPublica's investigation](https://www.propublica.org/article/machine-bias-risk-assessments-in-criminal-sentencing) uncovered bias against African Americans in software used for risk assessment in courtrooms (including here in the state of  Wisconsin), which judges can use to inform their decisions.
* Cathy O'Neil published [*Weapons of Math Destruction*](https://www.amazon.com/Weapons-Math-Destruction-Increases-Inequality/dp/0553418815), an excellent book warning about a world run by unregulated, opaque algorithms.
* The pre-Trump White House released a [report](https://obamawhitehouse.archives.gov/sites/default/files/whitehouse_files/microsites/ostp/NSTC/preparing_for_the_future_of_ai.pdf) on AI that explicitly warned about encoding discrimination in automated decision-making.

And this is just part of the popular coverage of algorithmic fairness.
In our unpopular (academic) world, the action has been primarily in the machine learning arena, where researchers have been studying ways to learn fair classifiers, for certain definitions of fairness.



## Unfair programs? We'll fairify them for you!

While algorithmic unfairness is an alarming
issue with potentially large-scale negative effects,
I believe that the move to algorithmic decision-making
has a silver lining: We can rigorously
reason about programs, debug them, and fix them.

In our work, we went after the following problem:
Say we're given a program that decides
whether to hire a job applicant
that is unfair (more on what that means in a bit).
By program I mean a piece of code that
is maybe a machine learning model, a
script distilled from the wisdom of a VP,
an SQL query written by a data scientist, whatever!
Our view is that such a program is probably
not designed to be blatantly unfair.
So what we'd like to do is to tweak it a little
bit and make it fair---I like to (unofficially)
call this process *fairification*.

The main question that I've avoided
so far is *how do you formalize fairness?!*
This is a deep philosophical problem.
But computer scientists love to formalize
the unformalizable!
Recent work in the area has proposed
several definitions.
Let's look at the one from [Feldman et al.](https://arxiv.org/pdf/1412.3756.pdf),
which formalizes  the 80--20
rule of thumb from the Equality of Employment
Commission here in the US:

{% math %}
\frac{Pr [hire | minority]}
{Pr [hire | \neg minority]} > 0.8
{% endmath %}

The intuition is simple: the probability
of hiring from the minority applicant pool
is at least 80% that of hiring from the non-minority pool---assuming a binary split
of the population.

OK, great. We've fully formalized
fairness, leaving no room for philosophy.

For an ultra-simple illustration,
say the program we have is the following:
It only takes one thing about the applicant,
the rank of the college they attended.
If the applicant attended a top-ten
school, then, good for them, they get hired!

```python
def hire(urank):
   return 1 <= urank <= 10
```

Is this program fair?
Well, it depends on the population!
We will represent the population as
a probabilistic model:
10% of the population are minorities;
non-minorities go to schools ranked 10
on average; minorities go to schools
ranked 15 on average (here {%m%}min{%em%}
is 1 for minority and 0 for non-minority).

{% math %}
min \sim Bernoulli(0.1)\\
urank \sim Gaussian(10 + 5*min, 10)
{% endmath %}

With this population model,
this program is unfair; on the above
fairness definition, the ratio
evaluates to ~0.6.
How do we fix it?

Our approach proceeds like this:
First, we characterize a class
of programs using a *sketch*
(I talked about sketches in the last [post]({{ site.baseurl }} {% post_url 2017-04-24-synthesis-primer%})).
One possible sketch here is the following,
where ```??``` are unknowns.

```python
def hire(urank):
   return ?? <= urank <= ??
```

Essentially, the sketch characterizes
a family of programs
(ML people call this a hypothesis class).
In this case, we've knocked out
the constants in the program and
we're hoping to replace them with new ones
to make it fair.
The same idea can be extended to not only
constants, but also instructions and branching.
The sketch encodes our *repair model*,
the various ways in which we can tweak
the original program.

Now, we want to find a completion of this
sketch such that
* 1) The completion is *semantically close* to the original program.
(Semantic closeness  just means that
  the two programs agree on most inputs.)
* 2) The completion is fair according
to the definition above.

The idea is that we want to give the program a small *nudge* to make it fair.
One possible completion is the following:
```python
def hire(urank):
   return 1 <= urank <= 15
```
This program happens to be fair, per the above definition, and is semantically close to the original program.
In a sense, we kept increasing the upper bound on the college ranking until we got a fair program. Our tool would find such completion.

Before I discuss how we actually do the *fairifcation*,
I have to state that I do not claim this is the best *debiasing* of our program or
that that fairness conditions I used in this example is the most desirable in this setting.
I simply intended this combination for illustration.

## Fairification with program synthesis

The approach we used is a new method for
program synthesis.
Most work in the program synthesis literature
attempts to find a program that satisfies a specification.
Here, our specification is a probabilistic one.
Our technique uses SMT solvers, fancy data structures,
and a sprinkle of statistical learning theory for good measure.
It traverses the space of programs
and finds fair programs that are close to the original unfair one.
Our technique can take an arbitrary fairness
definition in a syntactic language that
is expressions over probabilities of events, like the
80--20 rule we saw above.


For details, I invite you to read our [paper](http://pages.cs.wisc.edu/~aws/papers/fatml16.pdf).
For a quick synthesis primer, I invite
you to read an earlier [post]({{ site.baseurl }} {% post_url 2017-04-24-synthesis-primer%}).

## Looking forward

Most of the recent works have focused on unfairness in automation of bureaucratic processes, like loans, hiring, and others.
But fairness and unfairness extend to any other area where we interact with software. In the near future, it appears that  we'll be interacting with robots, self-driving cars, and other autonomous agents. What does fairness mean there?

I believe there's lots of interesting and important
work to be done by the programming languages
and verification communities on the issue
of fair programs:
*How do we debug unfair programs?
There are lots of algorithms introduced
for fair classification; can we build verified implementations? Can we build programming-language support for reasoning
about fairness in data analysis environments?*
