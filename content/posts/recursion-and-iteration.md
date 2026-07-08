+++
title = "Recursion and Iteration"
date = 2022-07-08T22:19:45-07:00

[taxonomies]
categories = ["CS"]
tags = ["algorithms", "recursion", "complexity"]
+++

Recursion and iteration are interchangeable. Any loop can be rewritten as recursion, and any recursion can be rewritten as a loop using an explicit stack. The difference is cost. Recursion consumes the call stack and adds per-call overhead. Iteration does neither. The right choice depends on the problem structure and the language.

<!--more-->

## When each fits

Recursion fits problems with recursive structure: trees, divide-and-conquer (merge sort, quicksort), and backtracking (N-queens, maze solving). The code mirrors the problem's shape. Iteration fits linear processes: scanning an array, accumulating a sum. Forcing recursion onto a linear problem adds overhead and risks stack overflow for no gain.

Python sets a recursion limit near 1000. Deep recursion raises `RecursionError`. C and Scheme optimize tail recursion into loops through tail-call optimization. Python and Java do not. In those languages, a tail-recursive function is strictly slower than the equivalent loop.

## The recursion tree

Recursive time complexity follows from the recursion tree. Each node is one call. Count the nodes.

For a single recursive call that halves the input, the depth is log n and each level has one node: O(log n). For two recursive calls that each halve the input, each level doubles the node count. The tree holds 1 + 2 + 4 + ... + 2^(log n) = 2n - 1 nodes: O(n). The branching factor, not the depth alone, determines the cost.

## The power function

Compute x^n. Four implementations show how recursion structure changes complexity.

**Loop.** Multiply x by itself n times.

```python
def power_loop(x, n):
    result = 1
    for _ in range(n):
        result *= x
    return result
```

O(n). No recursion overhead. The straightforward choice for this problem.

**Linear recursion.** Reduce n by 1 each call.

```python
def power_linear(x, n):
    if n == 0:
        return 1
    return power_linear(x, n - 1) * x
```

O(n). n calls, constant work each. Same complexity as the loop, but with call overhead and a stack depth of n. Recursion buys nothing here.

**Redundant branching recursion.** Halve n, but call twice.

```python
def power_redundant(x, n):
    if n == 0:
        return 1
    if n % 2 == 1:
        return power_redundant(x, n // 2) * power_redundant(x, n // 2) * x
    return power_redundant(x, n // 2) * power_redundant(x, n // 2)
```

The recursion tree branches two ways at each level. This looks like an optimization because the argument halves, but calling the function twice recomputes the entire subtree. The branching factor cancels the halving. Complexity stays O(n).

**Store the result.** Compute once, reuse.

```python
def power(x, n):
    if n == 0:
        return 1
    temp = power(x, n // 2)
    if n % 2 == 1:
        return temp * temp * x
    return temp * temp
```

Single recursive call, halving input. Depth log n, one node per level: O(log n). The win comes from collapsing two calls into one stored result.

## Redundant computation: Fibonacci

Fibonacci shows the cost of redundancy at its worst.

**Naive recursion.**

```python
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)
```

Each call spawns two. The tree grows as O(φ^n) where φ ≈ 1.618. fib(50) is already intractable. The two subproblems overlap: fib(n-2) is computed inside fib(n-1) and then again on its own.

**Memoization.** Store each result once.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n < 2:
        return n
    return fib_memo(n - 1) + fib_memo(n - 2)
```

Each value computes once. n distinct subproblems, constant work each: O(n).

**Iteration.** The same result without recursion.

```python
def fib_iter(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

O(n) time, O(1) space, no stack. For Fibonacci, iteration is the natural choice.
