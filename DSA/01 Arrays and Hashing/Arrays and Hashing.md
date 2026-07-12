---
tags: [dsa, pattern, arrays, hashing]
---

# Arrays & Hashing

The foundational pattern: trade space for time by using a **hash map (`dict`)** or **hash set (`set`)** to turn `O(n)`/`O(n^2)` lookups into `O(1)` lookups.

## When to reach for it
- You need to check "have I seen this before?" → `set`
- You need to count frequencies / group items by a key → `dict` (often `collections.Counter` or `collections.defaultdict`)
- You need to find a complement/pair that satisfies a condition (e.g. `target - num`) → `dict` mapping value → index/count
- Problem mentions "duplicates", "frequency", "grouping", "anagram", "contains" → hashing is likely involved

## Complexity cheatsheet
- Hash set/map insert, lookup, delete: `O(1)` average, `O(n)` worst case
- Building a frequency map over `n` elements: `O(n)` time, `O(n)` space

## Core Python tools

```python
from collections import Counter, defaultdict

# Counter: frequency map in one line
count = Counter([1, 1, 2, 3])          # Counter({1: 2, 2: 1, 3: 1})
count.most_common(2)                    # [(1, 2), (2, 1)] - top k by frequency

# defaultdict: avoid "if key not in dict" boilerplate
groups = defaultdict(list)
groups["a"].append(1)                   # no KeyError even though "a" wasn't set yet
```

## Template 1 — Two Sum (complement lookup)

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    seen = {}  # value -> index
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```
`O(n)` time, `O(n)` space. The trick: check for the complement **before** inserting the current number (handles `nums[i] + nums[i] == target` correctly without matching an element with itself).

## Template 2 — Group by signature (Group Anagrams)

```python
def group_anagrams(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))          # canonical signature for an anagram group
        groups[key].append(s)
    return list(groups.values())
```
`O(n * k log k)` time where `k` = max string length (sorting each string). Alternative `O(n * k)` key: a 26-length character-count tuple instead of sorting.

## Template 3 — Top K Frequent (bucket sort variant)

```python
def top_k_frequent(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)
    # bucket[i] = list of numbers that appear exactly i times
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in count.items():
        buckets[freq].append(num)

    result = []
    for freq in range(len(buckets) - 1, 0, -1):
        for num in buckets[freq]:
            result.append(num)
            if len(result) == k:
                return result
    return result
```
`O(n)` time — beats the naive `O(n log n)` "sort by frequency" approach by using bucket sort (frequency is bounded by `n`, so buckets work).

## Template 4 — Product of Array Except Self (prefix × suffix, no division)

```python
def product_except_self(nums: list[int]) -> list[int]:
    n = len(nums)
    res = [1] * n

    prefix = 1
    for i in range(n):
        res[i] = prefix
        prefix *= nums[i]

    suffix = 1
    for i in range(n - 1, -1, -1):
        res[i] *= suffix
        suffix *= nums[i]

    return res
```
`O(n)` time, `O(1)` extra space (output array doesn't count). Two passes: left-to-right prefix products, then right-to-left suffix products multiplied in. See also [[Prefix Sum]].

## Template 5 — Encode/Decode & Longest Consecutive Sequence pattern

```python
def longest_consecutive(nums: list[int]) -> int:
    num_set = set(nums)
    longest = 0

    for num in num_set:
        # only start counting from the beginning of a sequence
        if num - 1 not in num_set:
            length = 1
            while num + length in num_set:
                length += 1
            longest = max(longest, length)

    return longest
```
`O(n)` time despite the nested `while` — each number is only ever visited as the start of a sequence once, so total work across all iterations is `O(n)`. The `num - 1 not in num_set` guard is what keeps this linear.

## Pitfalls
- Using a `list` for membership checks (`x in list`) is `O(n)` — always prefer `set`/`dict` for repeated lookups.
- Mutable objects (lists) can't be dict keys/set members — convert to `tuple` first (see anagram grouping above).
- `defaultdict` silently creates entries on read (`d[missing_key]` inserts `missing_key`) — can cause subtle bugs if you later iterate/check `len(d)`.
- Sorting-based grouping is `O(n log n)` — bucket/counting approaches (Template 3) beat it when frequency is bounded by `n`.

## Representative problems
Two Sum · Valid Anagram · Group Anagrams · Top K Frequent Elements · Product of Array Except Self · Valid Sudoku · Encode and Decode Strings · Longest Consecutive Sequence

## Related
- [[Prefix Sum]]
- [[DSA MOC]]
