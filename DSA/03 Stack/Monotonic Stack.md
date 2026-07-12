---
tags: [dsa, technique, stack]
---

# Monotonic Stack

A stack that is kept **strictly increasing or strictly decreasing** from bottom to top by popping elements that would violate the order before pushing a new one. This lets you find, for every element, the **next/previous greater or smaller element** in `O(n)` total instead of `O(n^2)`.

## When to reach for it
- "Next greater element" / "next smaller element" / "previous greater element" phrasing
- Problems about spans, "how many days until warmer temperature", stock spans
- Histogram-style problems (largest rectangle in histogram)
- Anything where you'd otherwise write a nested loop comparing each element to "the next one that beats it"

## Core idea
Walk left to right maintaining a stack of **indices** whose corresponding values are in monotonic order. For each new element, pop everything from the stack that it "beats" (resolving those popped elements' answers), then push itself.

## Template — Next Greater Element (decreasing stack)

```python
def next_greater_element(nums: list[int]) -> list[int]:
    n = len(nums)
    result = [-1] * n
    stack = []  # holds indices; nums[stack] is decreasing bottom to top

    for i in range(n):
        # current element is greater than what's on top of stack ->
        # it IS the "next greater element" for those stacked indices
        while stack and nums[i] > nums[stack[-1]]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result
```
`O(n)` time — looks like a nested loop, but each index is pushed once and popped at most once, so total work across the whole run is `O(n)`. `O(n)` space.

## Template — Daily Temperatures (distance, not value)

```python
def daily_temperatures(temperatures: list[int]) -> list[int]:
    n = len(temperatures)
    result = [0] * n
    stack = []  # indices, temperatures[stack] decreasing bottom to top

    for i, temp in enumerate(temperatures):
        while stack and temp > temperatures[stack[-1]]:
            prev_idx = stack.pop()
            result[prev_idx] = i - prev_idx   # distance instead of value
        stack.append(i)

    return result
```
Same skeleton as above — monotonic stack problems almost always share this exact loop shape; only what you store/compute on resolution changes.

## Template — Largest Rectangle in Histogram (increasing stack, harder variant)

```python
def largest_rectangle_area(heights: list[int]) -> int:
    stack = []  # indices, heights[stack] increasing bottom to top
    max_area = 0

    for i, h in enumerate(heights + [0]):  # sentinel 0 flushes the stack at the end
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    return max_area
```
`O(n)` time, `O(n)` space. The width calculation is the tricky part: when popping index `j`, its rectangle spans from just after the new stack top to `i - 1`.

## How to recognize increasing vs decreasing
- Want **next greater element** → keep stack **decreasing**, pop while `current > stack top`.
- Want **next smaller element** → keep stack **increasing**, pop while `current < stack top`.
- (Mirror the condition if you're scanning right-to-left for "previous" instead of "next".)

## Pitfalls
- Storing values instead of **indices** on the stack — you usually need the index to compute distance/width/position, not just the value.
- Getting the pop condition's inequality direction backwards (greater vs. smaller, strict vs. non-strict) — always sanity check with a tiny example.
- Forgetting elements left on the stack at the end have no "next greater" — they keep their default (`-1` or `0`) unless a sentinel value is appended (see histogram example).

## Representative problems
Daily Temperatures · Next Greater Element I/II · Largest Rectangle in Histogram · Car Fleet

## Related
- [[Stack]]
- [[DSA MOC]]
