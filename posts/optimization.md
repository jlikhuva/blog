# Unified Treatment of Discrete Optimization

## Motivation

Often, when people learn about discrete optimization, they do so without explicitly learning about discrete optimization. For instance, a typical undergraduate CS curriculum may cover greedy algorithms and dynamic programming as separate concepts. That same curriculum may also cover algorithms for decision making in a separate class. In short, it's quite rare for discrete optimization to be presented in a unified manner that shows how all the different methods are related. That is what we wish to do in this note. We wish to present unified treatment of principal, concepts, and algorithms that underlie discrete optimization.

The rest of this note proceeds as follows: we start by examining the foundational idea underlying all of discrete optimization — choosing. We then discuss different schemes for choosing and see how these schemes directly lead to greedy algorithms and dynamic programming. For each of these schemes, we'll discuss and present well crafted code solutions for many different examples. After that, we'll explore techniques for choosing when stochasticity is involved. This discussion will naturally lead to our next topic: reinforcement learning. We'll examine methods that allow us to make efficient log-term decisions under uncertainty.

Before we begin, we should note that while this note aims to be expansive, it is by no means comprehensive. At the end, we've included links to primary sources that examine individual section in much more depth

[Integer Programming and MIP not covered. Also don't cover optimization on combinatoric structures such as graphs]

.As always, if you find any bug (by which we mean an idea that is not clearly explained, or an incorrect code sample), feel free to open a pull request with your suggested fix.  

## Decision Making

## Greedy Algorithms wrt an objective

## Dynamic Programming: Foundational Methods

## Decision Making Under Uncertainty

## Finite MDPs

## Dynamic Programming Methods in RL

## Temporal Difference Learning

## `n`-step Bootstrapping

## Planning & Learning with Tabular Methods
