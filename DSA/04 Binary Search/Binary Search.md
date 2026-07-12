---
tags: [dsa, pattern, binary-search]
---

# Binary Search

Halve the search space every step by exploiting **monotonicity** (something is sorted, or there's a boolean condition that flips exactly once). `O(log n)` instead of `O(n)`.

## When to reach for it
- Array is sorted (classic case)
- "Rotated sorted array" variants
- Search space is a **range of possible answers** rather than array indices — "binary search on the answer" (see below)
- Any condition of the form "find the boundary where `condition(x)` flips from False to True"

## Template 1 — Classic binary search

```python
def binary_search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2   # avoids overflow (not a Python concern, but idiomatic)
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```
`O(log n)` time, `O(1)` space. `left <= right` (not `<`) because with a single-element range (`left == right`) there's still a value to check.

## Template 2 — Search in Rotated Sorted Array

```python
def search_rotated(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid

        if nums[left] <= nums[mid]:          # left half is sorted
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:                                 # right half is sorted
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1
```
`O(log n)` time. Key idea: **at least one half is always properly sorted** — determine which half, then check if the target lies within that half's range to decide which side to discard.

## Template 3 — Binary search on the answer (find minimum/maximum feasible value)

Used when you're not searching an array but a **range of possible answers**, and you have a `feasible(x)` check that's monotonic (False...False, True...True).

```python
def min_eating_speed(piles: list[int], h: int) -> int:
    def hours_needed(speed: int) -> int:
        return sum((pile + speed - 1) // speed for pile in piles)  # ceil division

    left, right = 1, max(piles)
    result = right

    while left <= right:
        speed = (left + right) // 2
        if hours_needed(speed) <= h:   # feasible: try to go slower (smaller answer)
            result = speed
            right = speed - 1
        else:                          # infeasible: must go faster
            left = speed + 1

    return result
```
`O(n log m)` time where `m` = range of possible answers, `n` = cost of the feasibility check. **Pattern to recognize**: "minimize the maximum X" or "maximize the minimum X" subject to a constraint → binary search on the answer, with a helper function that checks feasibility for a candidate answer.

## Template 4 — Leftmost/rightmost insertion point (`bisect` module)

```python
import bisect

bisect.bisect_left(nums, target)    # leftmost index where target could be inserted
bisect.bisect_right(nums, target)   # rightmost index where target could be inserted
bisect.insort(nums, target)         # insert while maintaining sorted order (O(n) due to shifting)
```
Know these exist — in an interview you may need to implement the logic by hand, but for take-home/non-interview code, use `bisect`.

## Pitfalls
- `mid = (left + right) // 2` vs `left + (right - left) // 2` — identical in Python (no integer overflow), but the second is the habit to have for languages that do overflow.
- Off-by-one in loop condition (`<` vs `<=`) and in updating `left`/`right` (`mid` vs `mid ± 1`) — the #1 source of bugs. Trace through a 1-2 element array by hand to sanity check.
- Forgetting binary search requires **monotonicity**, not literal sortedness — rotated arrays and "binary search on answer" problems don't look sorted at first glance but still qualify.
- Infinite loops from not shrinking the range (e.g., `left = mid` instead of `left = mid + 1` when `mid` is proven not to be the answer).

## Representative problems
Binary Search · Search a 2D Matrix · Koko Eating Bananas · Find Minimum in Rotated Sorted Array · Search in Rotated Sorted Array · Time Based Key-Value Store · Median of Two Sorted Arrays

## Related
- [[Divide and Conquer]]
- [[DSA MOC]]
