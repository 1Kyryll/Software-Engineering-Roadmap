---
tags: [dsa, pattern, arrays, strings]
---

# Sliding Window

A specialization of [[Two Pointers]] where both pointers move in the **same direction**, maintaining a contiguous "window" `[left, right]` over an array/string. Avoids recomputing overlapping work, turning `O(n * k)` or `O(n^2)` brute force into `O(n)`.

## When to reach for it
- "Contiguous subarray/substring" is in the problem statement
- Fixed-size window: "subarray of size k" → maximize/minimize/find something
- Variable-size window: "longest/shortest substring satisfying condition X"
- You'd otherwise recompute a sum/count/frequency map from scratch for every subarray

## Two flavors

**1. Fixed-size window** — width `k` never changes; slide it across the array.
**2. Variable-size window** — grow `right` to include more, shrink `left` to fix a violated constraint.

## Template 1 — Fixed-size window (max sum subarray of size k)

```python
def max_subarray_sum(nums: list[int], k: int) -> int:
    window_sum = sum(nums[:k])
    best = window_sum

    for right in range(k, len(nums)):
        window_sum += nums[right] - nums[right - k]   # add new, remove oldest
        best = max(best, window_sum)

    return best
```
`O(n)` time, `O(1)` space. The key trick: update the running sum incrementally (`+new, -old`) instead of re-summing the window every time.

## Template 2 — Variable-size window (longest substring without repeating characters)

```python
def length_of_longest_substring(s: str) -> int:
    seen = set()
    left = 0
    best = 0

    for right in range(len(s)):
        while s[right] in seen:          # shrink until window is valid again
            seen.remove(s[left])
            left += 1
        seen.add(s[right])
        best = max(best, right - left + 1)

    return best
```
`O(n)` time — `left` and `right` each move forward at most `n` times total across the whole run (not `n` times *per iteration*), so it's linear despite the nested `while`. `O(min(n, alphabet size))` space.

## Template 3 — Variable-size window with a frequency map (minimum window substring)

```python
from collections import Counter

def min_window(s: str, t: str) -> str:
    if not t or not s:
        return ""

    need = Counter(t)
    missing = len(t)          # total chars still needed (with multiplicity)
    left = 0
    best_len, best_left = float('inf'), 0

    for right, char in enumerate(s):
        if need[char] > 0:
            missing -= 1
        need[char] -= 1

        while missing == 0:    # window is currently valid -> try to shrink
            if right - left + 1 < best_len:
                best_len, best_left = right - left + 1, left

            need[s[left]] += 1
            if need[s[left]] > 0:
                missing += 1   # removing this char breaks validity
            left += 1

    return "" if best_len == float('inf') else s[best_left:best_left + best_len]
```
`O(n + m)` time (`m` = len(t)), `O(m)` space for the counter. This is the hardest common sliding window template — the `missing` counter tracks "how many more characters (with multiplicity) do we still need" so validity is checked in `O(1)` instead of rescanning the window.

## General skeleton to reason from

```python
def sliding_window(s):
    left = 0
    # window_state = ... (running sum / Counter / set)

    for right in range(len(s)):
        # 1. expand: add s[right] to window_state

        while window_is_invalid():   # or "while window_state satisfies target condition"
            # 2. shrink: remove s[left] from window_state
            left += 1

        # 3. update answer using current window [left, right]

    return answer
```

## Pitfalls
- Recomputing the window's sum/count from scratch every iteration — defeats the purpose; always update incrementally.
- Off-by-one on window length: it's `right - left + 1`, not `right - left`.
- For "at most K distinct" vs "exactly K distinct" problems: "exactly K" is often solved as `atMost(K) - atMost(K-1)`, not directly.
- Forgetting the window can shrink to empty (`left > right`) in edge cases — guard against it if computing something like an average.

## Representative problems
Best Time to Buy and Sell Stock · Longest Substring Without Repeating Characters · Longest Repeating Character Replacement · Permutation in String · Minimum Window Substring · Sliding Window Maximum (uses a monotonic deque — see [[Monotonic Stack]] for the related idea)

## Related
- [[Two Pointers]]
- [[Monotonic Stack]]
- [[DSA MOC]]
