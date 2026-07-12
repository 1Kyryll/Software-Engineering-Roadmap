---
tags: [dsa, technique, linked-list, two-pointers]
---

# Fast and Slow Pointers

A specialization of [[Two Pointers]] (same-direction flavor) where one pointer moves twice as fast as the other. Most useful when there's no random access (linked lists) so you can't just index into the middle or check `nums[i]` vs `nums[j]` directly. Also called "Floyd's Tortoise and Hare."

## When to reach for it
- Find the **middle** of a linked list without knowing its length up front
- Detect a **cycle** in a linked list
- Find the **start of a cycle**
- Detect a cycle in a value-chain that isn't literally a linked list (e.g. Happy Number — repeatedly applying a function can be modeled as traversing an implicit linked list)

## Template 1 — Find the middle of a linked list

```python
def middle_node(head: ListNode) -> ListNode:
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    return slow   # when fast reaches the end, slow is at the middle
```
`O(n)` time, `O(1)` space, single pass — beats the naive "count length, then walk length/2 steps" two-pass approach.

## Template 2 — Detect a cycle (Floyd's algorithm)

```python
def has_cycle(head: ListNode) -> bool:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```
**Why it works**: if there's a cycle, `fast` enters it first and, since it gains on `slow` by 1 node per step (relative speed), it's guaranteed to eventually land exactly on `slow` — it cannot "jump over" it because the gap closes by exactly 1 each step. If there's no cycle, `fast` simply hits `None` and the loop ends. `O(n)` time, `O(1)` space (vs. `O(n)` space if you used a `set` to track visited nodes).

## Template 3 — Find the start of the cycle

```python
def detect_cycle_start(head: ListNode) -> ListNode:
    slow = fast = head

    # Phase 1: determine if a cycle exists, find the meeting point
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            break
    else:
        return None   # no cycle

    # Phase 2: move one pointer back to head; advance both at same speed
    slow = head
    while slow is not fast:
        slow = slow.next
        fast = fast.next

    return slow   # they meet exactly at the cycle's start
```
`O(n)` time, `O(1)` space. **Why phase 2 works**: let the distance from head to cycle start be `a`, from cycle start to meeting point be `b`. The math (derived from `fast` having traveled exactly `2×` the distance `slow` has) works out so that walking `a` more steps from the meeting point lands exactly on the cycle start — which is exactly what resetting one pointer to `head` and advancing both by 1 achieves. (Worth being able to sketch this on a whiteboard, but don't stress about re-deriving the algebra live — knowing the two-phase structure is what matters.)

## Template 4 — Cycle detection on an implicit chain (Happy Number)

```python
def is_happy(n: int) -> bool:
    def next_value(x: int) -> int:
        return sum(int(d) ** 2 for d in str(x))

    slow = fast = n
    while True:
        slow = next_value(slow)
        fast = next_value(next_value(fast))
        if slow == fast:
            break

    return slow == 1
```
Same Floyd's algorithm, applied to a sequence defined by repeated function application instead of literal `.next` pointers — recognizing "this is secretly a linked list" is the trick.

## Pitfalls
- Checking `fast and fast.next` (not just `fast`) in the loop condition — `fast.next.next` would otherwise throw on the last node.
- Using `==` instead of `is` when comparing nodes (value equality vs identity — for cycle detection you want identity).
- Forgetting phase 2 of "find cycle start" requires resetting one pointer to `head`, not to the meeting point.

## Representative problems
Linked List Cycle · Linked List Cycle II · Middle of the Linked List · Happy Number · Reorder List (uses find-middle + reverse + merge together) · Palindrome Linked List

## Related
- [[Two Pointers]]
- [[Linked List]]
- [[DSA MOC]]
