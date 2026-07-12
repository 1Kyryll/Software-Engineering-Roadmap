---
tags: [dsa, technique, arrays]
---

# Prefix Sum

Precompute cumulative sums so that **any range-sum query becomes `O(1)`** instead of `O(n)`.

## Core idea
```
prefix[i] = nums[0] + nums[1] + ... + nums[i-1]
```
Then the sum of `nums[l..r]` (inclusive) is `prefix[r+1] - prefix[l]`.

## When to reach for it
- Repeated "sum of subarray from index `l` to `r`" queries
- "Number of subarrays that sum to `k`" → combine with a hash map (see below)
- Problems that compute something incrementally left-to-right and need to compare "state so far" (e.g. product except self, running balance)

## Template 1 — Range sum queries

```python
def build_prefix(nums: list[int]) -> list[int]:
    prefix = [0] * (len(nums) + 1)
    for i, num in enumerate(nums):
        prefix[i + 1] = prefix[i] + num
    return prefix

def range_sum(prefix: list[int], l: int, r: int) -> int:
    # inclusive sum of nums[l..r]
    return prefix[r + 1] - prefix[l]
```
Build: `O(n)` time/space. Each query after that: `O(1)`.

## Template 2 — Subarray Sum Equals K (prefix sum + hash map)

```python
def subarray_sum(nums: list[int], k: int) -> int:
    count = 0
    running_sum = 0
    seen = {0: 1}  # prefix sum -> number of times seen (0 handles subarray starting at index 0)

    for num in nums:
        running_sum += num
        # if (running_sum - k) was seen before, that prefix..here slice sums to k
        count += seen.get(running_sum - k, 0)
        seen[running_sum] = seen.get(running_sum, 0) + 1

    return count
```
`O(n)` time, `O(n)` space. This is the pattern to recognize: "subarray sum == k" is almost always prefix sum + hashmap, not brute force `O(n^2)`.

## Pitfalls
- Off-by-one on the prefix array length (`len(nums) + 1` with `prefix[0] = 0` avoids special-casing `l == 0`).
- Forgetting to seed `seen = {0: 1}` in Template 2 — without it, subarrays starting at index 0 are undercounted.
- Prefix sum doesn't help if the array is mutated between queries (would need a Fenwick tree / segment tree instead — out of scope for NeetCode 150 but worth knowing exists).

## Representative problems
Range Sum Query (Immutable) · Subarray Sum Equals K · Product of Array Except Self (prefix × suffix variant)

## Related
- [[Arrays and Hashing]]
- [[DSA MOC]]
