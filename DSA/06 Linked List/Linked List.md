---
tags: [dsa, pattern, linked-list]
---

# Linked List

A chain of nodes, each pointing to the next (singly) or both next and previous (doubly). No random access (`O(n)` to reach index `i`), but `O(1)` insertion/deletion once you have a reference to the right node — the opposite tradeoff profile from arrays.

## Node definition

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

## When to reach for it
- Problem explicitly gives a linked list
- "Reverse", "merge", "detect cycle", "find middle", "remove nth from end" phrasing
- Need `O(1)` insert/delete and don't need random access

## The dummy/sentinel node trick
Almost every linked-list-building problem gets simpler with a **dummy head** — a placeholder node before the real head. It eliminates special-casing "is this the first node?".

```python
dummy = ListNode()
tail = dummy
# ... build the list by doing tail.next = new_node; tail = tail.next ...
return dummy.next   # skip the dummy itself
```

## Template 1 — Reverse a Linked List (iterative)

```python
def reverse_list(head: ListNode) -> ListNode:
    prev = None
    curr = head

    while curr:
        next_node = curr.next   # save before overwriting
        curr.next = prev
        prev = curr
        curr = next_node

    return prev   # new head
```
`O(n)` time, `O(1)` space. The three-pointer shuffle (`prev`, `curr`, `next_node`) is the pattern to memorize — it recurs in "reverse in groups of k" and "reverse between positions m and n" variants.

Recursive version (trades `O(1)` space for `O(n)` call stack — good to know both):
```python
def reverse_list_recursive(head: ListNode) -> ListNode:
    if not head or not head.next:
        return head
    new_head = reverse_list_recursive(head.next)
    head.next.next = head
    head.next = None
    return new_head
```

## Template 2 — Merge Two Sorted Lists

```python
def merge_two_lists(l1: ListNode, l2: ListNode) -> ListNode:
    dummy = ListNode()
    tail = dummy

    while l1 and l2:
        if l1.val <= l2.val:
            tail.next = l1
            l1 = l1.next
        else:
            tail.next = l2
            l2 = l2.next
        tail = tail.next

    tail.next = l1 if l1 else l2   # attach whichever list has leftovers
    return dummy.next
```
`O(n + m)` time, `O(1)` extra space (rewiring existing nodes, not creating new ones).

## Template 3 — Remove Nth Node From End (one-pass, two pointers)

```python
def remove_nth_from_end(head: ListNode, n: int) -> ListNode:
    dummy = ListNode(next=head)
    fast = slow = dummy

    for _ in range(n):
        fast = fast.next        # advance fast n steps ahead

    while fast.next:            # move both until fast hits the end
        fast = fast.next
        slow = slow.next

    slow.next = slow.next.next  # slow is now right before the node to remove
    return dummy.next
```
`O(n)` time (single pass), `O(1)` space. Gap of `n` between `fast` and `slow` means when `fast` reaches the end, `slow` is exactly at the node before the target — avoids a first pass to count length.

## Template 4 — Detect and locate a cycle (Floyd's algorithm)
See [[Fast and Slow Pointers]] for the full derivation — this is the flagship linked-list use of that technique.

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

## Doubly linked lists
Each node also has `.prev`. Used for `O(1)` arbitrary removal (no need to scan to find the predecessor) — e.g. LRU Cache combines a doubly linked list (for O(1) reorder/evict) with a hash map (for O(1) lookup by key).

## Pitfalls
- Losing the reference to the rest of the list before rewiring (`curr.next = prev` before saving `next_node`) — always save `next_node` first.
- Off-by-one with dummy nodes — remember to `return dummy.next`, not `dummy`.
- Not handling `head is None` or single-node lists as edge cases.
- Comparing nodes with `==` vs `is` — for cycle detection you want `is` (same object), not value equality.

## Representative problems
Reverse Linked List · Merge Two Sorted Lists · Reorder List · Remove Nth Node From End of List · Copy List with Random Pointer · Add Two Numbers · Linked List Cycle · LRU Cache · Merge K Sorted Lists

## Related
- [[Fast and Slow Pointers]]
- [[Two Pointers]]
- [[DSA MOC]]
