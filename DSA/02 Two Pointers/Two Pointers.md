---
tags: [dsa, pattern, arrays]
---

# Two Pointers

Use two indices moving through a structure (usually a sorted array or string) to avoid nested loops, turning `O(n^2)` brute force into `O(n)`.

## When to reach for it
- Array is **sorted** (or can be sorted) and you're looking for a pair/triplet satisfying a condition
- Palindrome checking (compare from both ends inward)
- Merging two sorted structures
- "Container"/area problems (maximize something bounded by two indices)
- Removing duplicates / partitioning in-place

## Two flavors

**1. Opposite direction (converging)** — pointers start at both ends, move toward each other.
**2. Same direction (fast/slow)** — both pointers start together or nearby, move forward at different rates. See also [[Fast and Slow Pointers]] for the linked-list-specific version.

## Template 1 — Opposite direction: Two Sum II (sorted input)

```python
def two_sum_sorted(nums: list[int], target: int) -> list[int]:
    left, right = 0, len(nums) - 1

    while left < right:
        curr_sum = nums[left] + nums[right]
        if curr_sum == target:
            return [left, right]
        elif curr_sum < target:
            left += 1      # need a bigger sum -> move left pointer up
        else:
            right -= 1     # need a smaller sum -> move right pointer down

    return []
```
`O(n)` time, `O(1)` space. Works because the array is sorted: moving `left` only increases the sum, moving `right` only decreases it — no combination is ever skipped.

## Template 2 — Opposite direction: Container With Most Water

```python
def max_area(heights: list[int]) -> int:
    left, right = 0, len(heights) - 1
    best = 0

    while left < right:
        width = right - left
        height = min(heights[left], heights[right])
        best = max(best, width * height)

        # always move the pointer at the SHORTER wall —
        # moving the taller one can never increase the area (width shrinks, height capped by the short wall)
        if heights[left] < heights[right]:
            left += 1
        else:
            right -= 1

    return best
```
`O(n)` time, `O(1)` space. The key insight (interviewers love asking you to justify this): moving the taller pointer is provably never better.

## Template 3 — Three Sum (fix one, two-pointer the rest)

```python
def three_sum(nums: list[int]) -> list[list[int]]:
    nums.sort()
    res = []

    for i in range(len(nums)):
        if nums[i] > 0:
            break  # smallest number is positive -> no triplet can sum to 0
        if i > 0 and nums[i] == nums[i - 1]:
            continue  # skip duplicate "first" element

        left, right = i + 1, len(nums) - 1
        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total < 0:
                left += 1
            elif total > 0:
                right -= 1
            else:
                res.append([nums[i], nums[left], nums[right]])
                left += 1
                right -= 1
                while left < right and nums[left] == nums[left - 1]:
                    left += 1  # skip duplicate "second" elements

    return res
```
`O(n^2)` time (sort `O(n log n)` + `O(n)` outer loop × `O(n)` inner two-pointer scan), `O(1)` extra space excluding the sort/output. Pattern: **reduce k-sum to (k-1)-sum** by fixing one element and recursing/looping.

## Template 4 — Same direction: remove/compact in place

```python
def remove_duplicates(nums: list[int]) -> int:
    if not nums:
        return 0

    slow = 0  # boundary of the "clean" region
    for fast in range(1, len(nums)):
        if nums[fast] != nums[slow]:
            slow += 1
            nums[slow] = nums[fast]

    return slow + 1  # new length
```
`O(n)` time, `O(1)` space. `slow` tracks the write position, `fast` scans ahead — classic in-place compaction.

## Pitfalls
- Two pointers on an **unsorted** array usually doesn't work — sort first if order doesn't matter, or confirm the problem's structure justifies it.
- Forgetting to skip duplicates in problems like 3Sum leads to duplicate triplets in the output.
- Off-by-one on `while left < right` vs `while left <= right` — depends on whether the same index can pair with itself.

## Representative problems
Valid Palindrome · Two Sum II · 3Sum · Container With Most Water · Trapping Rain Water · Remove Duplicates from Sorted Array

## Related
- [[Fast and Slow Pointers]]
- [[DSA MOC]]
