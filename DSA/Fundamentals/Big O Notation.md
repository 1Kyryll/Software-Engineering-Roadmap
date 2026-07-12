---
tags: [dsa, fundamentals]
---

# Big O Notation

Big O describes how runtime/space **scales** with input size `n`, ignoring constants. It answers "what happens as `n → ∞`", not "how fast is this in milliseconds."

## Common complexities (best to worst)

| Notation     | Name         | Example                             |
| ------------ | ------------ | ----------------------------------- |
| `O(1)`       | Constant     | hash map lookup, array index access |
| `O(log n)`   | Logarithmic  | binary search                       |
| `O(n)`       | Linear       | single loop over array              |
| `O(n log n)` | Linearithmic | merge sort, sorting in general      |
| `O(n^2)`     | Quadratic    | nested loops, naive pair-checking   |
| `O(2^n)`     | Exponential  | naive recursive fibonacci, subsets  |
| `O(n!)`      | Factorial    | permutations                        |

## Rules of thumb
- Drop constants: `O(2n)` → `O(n)`.
- Drop lower-order terms: `O(n^2 + n)` → `O(n^2)`.
- Different input variables get different letters: iterating two arrays of different sizes is `O(n + m)`, not `O(n)`.
- **Amortized** complexity matters for things like Python `list.append` (`O(1)` amortized) or dynamic array resizing.
- Hash map/set operations are `O(1)` **average case**, `O(n)` worst case (hash collisions) — interviewers usually mean average case unless stated otherwise.

## Space complexity
Count extra memory used relative to input, not counting the input itself (unless you mutate/copy it):
- Recursion depth counts as space (call stack) — e.g., naive recursion on a linked list of length `n` is `O(n)` space.
- In-place algorithms (two pointers, most sorting done in-place) are `O(1)` extra space.

## Quick reference for Python built-ins
| Operation | Complexity |
|---|---|
| `list[i]` access | `O(1)` |
| `list.append` | `O(1)` amortized |
| `list.pop()` (end) | `O(1)` |
| `list.pop(0)` (front) | `O(n)` |
| `list.insert(i, x)` | `O(n)` |
| `x in list` | `O(n)` |
| `x in set` / `x in dict` | `O(1)` average |
| `sorted(list)` | `O(n log n)` |
| `dict/set add, get, delete` | `O(1)` average |
| `heapq.heappush/heappop` | `O(log n)` |
| `collections.deque` append/pop (both ends) | `O(1)` |

## Related
- [[DSA MOC]]
