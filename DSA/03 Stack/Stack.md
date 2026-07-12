---
tags: [dsa, pattern, stack]
---

# Stack

LIFO (last-in, first-out) structure. In Python, just use a `list` — `append`/`pop` from the end are both `O(1)`. (Don't use `list.pop(0)` or `insert(0, ...)` — those are `O(n)`; if you need FIFO or both-end operations, use `collections.deque`.)

## When to reach for it
- Matching/nesting problems (parentheses, tags, nested structures)
- "Undo" / backtrack-to-previous-state problems
- Evaluating expressions (RPN, calculators)
- Anything requiring "the most recent unmatched X" → see also [[Monotonic Stack]] for "the most recent unmatched X that's also increasing/decreasing"
- Recursion can often be rewritten iteratively using an explicit stack (useful to avoid recursion-depth limits)

## Template 1 — Valid Parentheses

```python
def is_valid(s: str) -> bool:
    pairs = {')': '(', ']': '[', '}': '{'}
    stack = []

    for char in s:
        if char in pairs:                    # closing bracket
            if not stack or stack[-1] != pairs[char]:
                return False
            stack.pop()
        else:                                 # opening bracket
            stack.append(char)

    return not stack   # must be fully matched
```
`O(n)` time, `O(n)` space. The canonical stack problem — every "does this nest correctly" question reduces to this.

## Template 2 — Min Stack (stack that also tracks its own minimum in O(1))

```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []  # min_stack[i] = min of stack[0..i]

    def push(self, val: int) -> None:
        self.stack.append(val)
        new_min = min(val, self.min_stack[-1] if self.min_stack else val)
        self.min_stack.append(new_min)

    def pop(self) -> None:
        self.stack.pop()
        self.min_stack.pop()

    def top(self) -> int:
        return self.stack[-1]

    def get_min(self) -> int:
        return self.min_stack[-1]
```
All operations `O(1)`. Trick: maintain a **parallel stack** where each entry is "the min so far including this element" — no recomputation needed on pop.

## Template 3 — Evaluate Reverse Polish Notation

```python
def eval_rpn(tokens: list[str]) -> int:
    stack = []
    ops = {
        '+': lambda a, b: a + b,
        '-': lambda a, b: a - b,
        '*': lambda a, b: a * b,
        '/': lambda a, b: int(a / b),  # truncate toward zero, per RPN spec
    }

    for token in tokens:
        if token in ops:
            b = stack.pop()   # note: b popped first (it's the second operand)
            a = stack.pop()
            stack.append(ops[token](a, b))
        else:
            stack.append(int(token))

    return stack[0]
```
`O(n)` time, `O(n)` space. Watch the pop order — for non-commutative ops (`-`, `/`) the first-popped value is the *right-hand* operand.

## Template 4 — Iterative DFS using an explicit stack

```python
def dfs_iterative(root):
    if not root:
        return []

    stack, result = [root], []
    while stack:
        node = stack.pop()
        result.append(node.val)
        # push right first so left is processed first (LIFO)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result
```
Turns recursive DFS into an explicit stack — useful when recursion depth could exceed Python's limit.

## Pitfalls
- Checking `if not stack` before popping — popping an empty stack raises `IndexError`.
- For RPN/expression evaluation, operand pop order matters for `-` and `/`.
- Python `list` as a stack is efficient only at the **end** — never use index 0.

## Representative problems
Valid Parentheses · Min Stack · Evaluate Reverse Polish Notation · Generate Parentheses (backtracking + stack-like reasoning) · Daily Temperatures (see [[Monotonic Stack]]) · Car Fleet

## Related
- [[Monotonic Stack]]
- [[DSA MOC]]
