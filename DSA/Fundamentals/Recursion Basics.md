---
tags: [dsa, fundamentals]
---

# Recursion Basics

Recursion = a function that calls itself on a smaller subproblem, plus a **base case** that stops the recursion.

## Anatomy

```python
def recurse(state):
    # 1. Base case(s) — smallest problem, answer known directly
    if is_base_case(state):
        return base_answer

    # 2. Recursive case — shrink the problem, trust the recursive call
    smaller_result = recurse(shrink(state))

    # 3. Combine — build this level's answer from the smaller result
    return combine(state, smaller_result)
```

## Mental model: trust the recursive call
Don't trace the whole call stack in your head. Assume `recurse(smaller_input)` already correctly solves the smaller problem — just figure out how to use that result to solve the current level. This is the single biggest unlock for writing recursive solutions quickly.

## Recursion → complexity
- Each call typically does `O(1)` or `O(k)` work outside the recursive call(s).
- Total time ≈ (number of calls) × (work per call).
- Call stack depth = extra **space** complexity (e.g. `O(n)` for linear recursion, `O(log n)` for a balanced binary recursion like binary search).

## Common failure modes
- **Missing/incorrect base case** → infinite recursion → `RecursionError` (Python's default recursion limit is ~1000).
- **Not shrinking the input** on every recursive call → infinite recursion.
- **Recomputing overlapping subproblems** → exponential blowup (classic naive Fibonacci is `O(2^n)`); fix with memoization → this is the bridge into Dynamic Programming.
- **Mutating shared state** (e.g. a shared list) across branches without undoing it — relevant later in backtracking.

## Fibonacci: naive vs memoized

```python
# Naive: O(2^n) time, O(n) space (call stack)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

# Memoized: O(n) time, O(n) space
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo)
    return memo[n]
```

## Where recursion shows up in NeetCode 150
- [[Divide and Conquer]] (merge sort, [[Binary Search]])
- [[Linked List]] problems (reverse, merge — can be done recursively or iteratively)
- Trees (nearly everything)
- Backtracking (subsets, permutations, combinations)
- 1-D / 2-D DP (recursion + memoization)

## Related
- [[DSA MOC]]
- [[Divide and Conquer]]
