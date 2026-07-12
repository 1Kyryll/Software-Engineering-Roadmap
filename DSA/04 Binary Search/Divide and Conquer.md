---
tags: [dsa, paradigm]
---

# Divide and Conquer

A problem-solving paradigm, not a single pattern: **split** the problem into smaller independent subproblems, **conquer** them recursively, then **combine** the results. [[Binary Search]] is the simplest example (split in half, recurse into one half, no combine step needed).

## Anatomy
1. **Divide** — split input into subparts (often halves)
2. **Conquer** — recursively solve each subpart
3. **Combine** — merge the subresults into the answer for the original problem

Contrast with [[Recursion Basics]]: all divide and conquer is recursive, but not all recursion divides the problem into independent parts (e.g. simple linear recursion like factorial doesn't "combine" multiple branches).

## Template — Merge Sort

```python
def merge_sort(nums: list[int]) -> list[int]:
    if len(nums) <= 1:
        return nums

    mid = len(nums) // 2
    left = merge_sort(nums[:mid])     # divide + conquer (left half)
    right = merge_sort(nums[mid:])    # divide + conquer (right half)
    return merge(left, right)          # combine

def merge(left: list[int], right: list[int]) -> list[int]:
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```
`O(n log n)` time (log n levels of recursion, O(n) work merging at each level), `O(n)` space. The canonical divide-and-conquer sort — contrast with quicksort, which divides unevenly around a pivot and combines trivially (no merge step, but a costly partition step).

## Template — Quick Sort (divide unevenly via a pivot)

```python
def quick_sort(nums: list[int]) -> list[int]:
    if len(nums) <= 1:
        return nums

    pivot = nums[len(nums) // 2]
    left = [x for x in nums if x < pivot]
    mid = [x for x in nums if x == pivot]
    right = [x for x in nums if x > pivot]
    return quick_sort(left) + mid + quick_sort(right)
```
`O(n log n)` average, `O(n^2)` worst case (bad pivot choices, e.g. already-sorted input with a naive pivot). This simple version isn't in-place; the in-place partition version (Lomuto/Hoare) is what's usually asked about for follow-up "can you do it in-place" questions.

## How to recognize a divide-and-conquer problem
- The problem on the full input can be answered by combining answers on two (or more) independent halves
- Naive solution is `O(n^2)` but the structure suggests recursion could cut it to `O(n log n)`
- Explicitly involves sorting, or "find the k-th something", or merging sorted structures

## Complexity: the Master Theorem (intuition, not memorization)
For `T(n) = a*T(n/b) + O(n^d)`:
- If splitting into `a` subproblems of size `n/b`, and `d` = cost of divide/combine step:
  - `a = 2, b = 2, d = 1` (merge sort) → `O(n log n)`
  - `a = 1, b = 2, d = 0` (binary search: one subproblem, O(1) combine) → `O(log n)`
- Rule of thumb: the more subproblems (`a`) relative to how much smaller each gets (`b`), the worse the complexity — this is why naive multi-branch recursion without memoization explodes exponentially (see [[Recursion Basics]]).

## Pitfalls
- Forgetting the base case → infinite recursion.
- Doing `O(n)` extra work at each level unnecessarily (e.g., slicing lists in Python is itself `O(n)` — merge sort's `nums[:mid]` slicing adds a hidden constant factor; for performance-critical code, pass indices instead of new list slices).
- Confusing divide-and-conquer (independent subproblems, combined once) with dynamic programming (overlapping subproblems, need memoization) — if subproblems overlap, D&C alone leads to exponential blowup; that's the DP bridge.

## Representative problems
Merge Sort · Quick Sort · Binary Search · Median of Two Sorted Arrays · Maximum Subarray (D&C variant of Kadane's) · Count of Smaller Numbers After Self

## Related
- [[Binary Search]]
- [[Recursion Basics]]
- [[DSA MOC]]
